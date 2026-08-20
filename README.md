# pkr.img

Photo of a chip stack in, settlement out. A web app for closing out a home poker
game: every player photographs their chips, and the host gets a list of who pays whom.

Next.js and FastAPI. The web app works end to end. The vision half is honest about
where it stands, see [Status](#status).

## The problem

Settling a home game is a counting problem followed by a graph problem, and both get
done by hand at 1am. Everyone has a pile of mixed denominations, someone tallies them
on paper, and then the table works out the transfers. That usually ends up as more
payments than it needs to be, because people settle pairwise as they remember debts.

Both halves are worth automating, and they are independent, which is why the app is
useful before the camera part is finished.

## Settlement

Given each player's cash out and a fixed buy in, profit and loss is one subtraction.
The transfers are the interesting part.

The naive version has everyone who lost pay everyone who won. The implementation sorts
debtors by how much they owe and creditors by how much they are owed, then repeatedly
pays the largest debtor into the largest creditor for the smaller of the two amounts.
Every payment zeroes out at least one of the pair, so `n` players never need more than
`n - 1` transfers.

That is not provably the minimum for every arrangement, and finding that minimum in
general is a harder problem than this deserves. The `n - 1` bound is the property that
matters at a kitchen table: nobody sends four Venmos when one would do.

Only a player's most recent submission counts, so a bad photo is fixed by uploading
another one.

## Vision

`api/cv_service.py` is the pipeline: YOLOv8-seg finds chip regions, regions are
clustered into stacks by proximity, chips within a stack are counted by finding the
seams between them with edge detection, and each stack is classified by dominant color
in HSV to get a denomination. HoughCircles is the fallback when segmentation returns
nothing.

**No trained weights are in this repository.** Without them the service loads
stock `yolov8n-seg.pt`, which is trained on COCO and does not know what a poker chip
is, so an upload records a total of zero rather than failing. Labeling and fine tuning
do produce usable masks, but turning masks into reliable values across angles and
occlusion is the open problem. Details and how to point the service at your own model
are in [docs/cv-pipeline.md](docs/cv-pipeline.md).

## Running it

Backend, in a virtual environment. Some of these pin against a base conda environment
badly, so an isolated one saves a fight:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r api/requirements.txt
uvicorn api.main:app --reload          # http://127.0.0.1:8000
```

Frontend:

```bash
cd web && npm install && npm run dev   # http://localhost:3000
```

Check the API is up with `curl http://127.0.0.1:8000/health`, which returns
`{"status": "ok"}`.

## How it fits together

```
api/
  main.py         routes: create, join, upload, end
  models.py       Party, Player, Submission
  cv_service.py   the chip pipeline
  db.py           SQLAlchemy session, SQLite
web/app/
  page.tsx                     host or join
  party/[code]/host/page.tsx   host dashboard: who joined, who submitted, end game
  party/[code]/join/page.tsx   join by code
  party/[code]/upload/page.tsx photo upload, same page for everyone
  party/[code]/page.tsx        party dashboard and results
```

A few decisions worth naming:

**The host is also a player.** Creating a party registers you as its first player, so
the host uploads chips like everyone else and there is one upload page rather than two.

**Role is client side.** `localStorage` holds `role:{partyCode}`, and the upload page
redirects to the host or party dashboard based on it. Cheap, and it means no auth in
the MVP.

**Dashboards poll every two seconds.** Not WebSockets. Polling is a few lines, and at
the size of a poker table it is indistinguishable.

## Status

The MVP works: parties, joins, uploads, and settlement run end to end on SQLite.

What is not done, in order: turning masks into values reliably, which is the whole
point of the app, then pushing dashboard updates when a player resubmits instead of
polling for them. Auth, Postgres, and a deployment are further out and not the
interesting part.
