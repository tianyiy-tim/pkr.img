# The chip pipeline

What `api/cv_service.py` does, what it needs that is not in this repository, and how to
point it at your own model.

## The contract

One image in, a value out. The service returns the same shape whether it succeeded,
fell back, or found nothing:

```json
{
  "total_cents": 5000,
  "breakdown": { "denom_500": 10 },
  "meta": {
    "model": "yolov8-seg",
    "confidence": 0.85,
    "notes": "Detected 10 chip regions, 2 stacks"
  }
}
```

`main.py` stores `total_cents` on the submission and keeps the breakdown as JSON, so
settlement never has to know how the number was produced. The upload route catches
anything the service throws and records zero, on the reasoning that a failed photo
should not take the party down with it. The tradeoff is that a silent zero looks like a
player who lost everything, which is a thing to fix before this is in front of anyone
who cares.

## The stages

1. **Detect.** YOLOv8-seg returns segmentation masks for chip regions. Masks rather
   than boxes, because chips in a stack overlap and a box around one contains most of
   its neighbors.
2. **Cluster into stacks.** Detected chips are grouped by spatial proximity, since the
   thing a person makes on a table is stacks, not loose chips.
3. **Count within a stack.** A stack seen from the side is a striped cylinder, so
   counting chips means counting seams: edge detection over the stack region, then
   counting the distinct horizontal bands the seams separate.
4. **Classify.** Each stack gets a dominant color in HSV, which maps to a
   denomination.
5. **Total.** Counts times denominations, summed.

If step 1 returns nothing, HoughCircles looks for circular objects instead. That is a
worse detector and it is only there so a photo of chips face up produces something
rather than nothing.

## What is missing

The repository has no trained weights. `ChipCVService` falls back to stock
`yolov8n-seg.pt`, which is trained on COCO and has no chip class, so out of the box it
detects nothing and every upload records zero.

Getting real numbers needs a model fine tuned on chip photos: collect images across the
angles and lighting a real table produces, label segmentation masks with something like
Label Studio or Roboflow, train YOLOv8-seg on them, and pass the result in.

```python
from api.cv_service import ChipCVService

service = ChipCVService(
    model_path="path/to/chip_model.pt",
    denomination_config={"red": 500, "blue": 1000, "green": 2500, "black": 5000},
)
```

Denominations are in cents and default to that mapping. The HSV ranges in
`color_ranges` are tuned to one particular set of chips and will need adjusting for
another.

To try the service directly:

```python
from api.cv_service import process_chip_image
print(process_chip_image("path/to/photo.jpg"))
```

## The hard part

Detection is not the bottleneck. Fine tuning produces usable masks. Turning masks into
values is where this stands, and the reasons are specific:

- **Occlusion.** A stack hides all but its rim. Counting seams works until a chip's
  seam is not visible from the camera's angle.
- **Angle.** Seam spacing in pixels depends on how far the stack is tilted away, so a
  count calibrated at one angle over or under counts at another.
- **Color under real light.** HSV thresholds that separate red from black on a kitchen
  table stop separating them under a warm bulb.

Any of these can be worked around with a calibration reference in frame, which is
probably where this goes next.
