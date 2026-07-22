# Varda

**Can I be here? Can I do this here?** Varda answers those questions for
wherever you're standing — who manages the ground, whether you're welcome,
and what the rules are for what you want to do.

*Varda* is Old Norse for cairn, the stacked stones that mark a trail; the
root also means *guardian*. It's the working sibling of
[Cairn](https://cairnapp.org), which is free and stays free.

## Status

Early prototype — the first heartbeat. One region (Oregon's Willamette
Valley and Cascades), three questions:

- **Be here** — public access, from the ground up
- **Ride a bike** — trail rules by land manager and live federal data
- **Drive** — Motor Vehicle Use Map designations, classes, and seasons

## How it works

One HTML file, no server, no accounts, no API keys. The page asks live
government services directly from your browser:

- **[USGS Protected Areas Database (PAD-US)](https://www.usgs.gov/programs/gap-analysis-project/science/pad-us-data-overview)** — who manages the ground, designations, access codes
- **[US Forest Service Enterprise Data Warehouse](https://data-usfs.hub.arcgis.com/)** — trail managed uses and Motor Vehicle Use Map roads
- **[OpenStreetMap](https://www.openstreetmap.org/)** — place names, so a bad GPS fix announces itself

Where live data ends, a small hand-maintained rules table takes over. Every
row carries its source, a link, and a verified-on date. When we don't have
a verified answer, the app says so and points you to the office that does —
that's a designed screen, not an error.

## The honesty rules

- Answers carry their confidence: verified, hedged, or unknown.
- An absent restriction means *no recorded restriction*, never "allowed."
- Signs on the ground outrank the database, always.
- Name every source. Date every rule. End at the ground.

## What it is not

Varda informs; it never claims authority over the ground. It is not legal
advice, and it can't see today's closures, gates, or fire orders. Always
confirm rules and conditions locally — the land's own stewards know it
best.
