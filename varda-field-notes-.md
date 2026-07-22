# Varda Field Notes — Data Plumbing & First Heartbeat

Companion to the Cairn data-plumbing handoff. Cairn's lessons all still hold
(mirror fallbacks, silent enrichment, hedged copy, credit everyone). This
document covers what Varda added on top, and what five real places taught us
when we pointed live queries at them: Sandy Ridge (BLM bike park), Beef Basin
Road (Forest Service, Utah), McDonald-Dunn Research Forest (Oregon State
University), Cascade Pass (North Cascades NP), and a living-room couch.

Everything below was learned by doing, most of it the hard way. That's what
makes it worth writing down.

**v1 scope: four questions — can I be here, bikes on trails, motor
vehicles on roads, and what lives here.** Access and species both came
from data already in hand: PAD-US carries a public-access code on every
record, and IPaC answers the species question outright.
Rock collecting was researched in full and deliberately cut from v1: its
rules live almost entirely off-data (per-forest discretion, unpublished
policies, mining claims), so it would have been the only activity whose
answers rested on unverified rows. The research is kept in section 8 for
when collection comes back.

---

## 1. The pipes

All free, no keys, called straight from the browser. Same ArcGIS query
pattern as Cairn's PAD-US pipe throughout.

| Source | Endpoint | Used for | Field-name casing |
|---|---|---|---|
| PAD-US (Manager_Name) | `services.arcgis.com/v01gqwM5QqNysAAi/.../Manager_Name/FeatureServer/0/query` | Who manages the ground; designations; access codes | `CamelCase` (`Unit_Nm`, `Mang_Name`, `Pub_Access`) |
| USFS Trails (EDW) | `apps.fs.usda.gov/arcx/rest/services/EDW/EDW_TrailNFSPublish_01/MapServer/0/query` | Per-trail managed uses: bike, hiker, horse, motorcycle | `lowercase` (`trail_name`, `bicycle_managed`) |
| USFS MVUM Roads (EDW) | `apps.fs.usda.gov/arcx/rest/services/EDW/EDW_MVUM_01/MapServer/1/query` | Legal motor-vehicle designations, classes, seasons | `lowercase` (`name`, `seasonal`, `jurisdiction`) |
| USFWS IPaC (Location API) | `ipac.ecosphere.fws.gov/location/api/resources` | Listed species, birds, critical habitat, refuges, field office | `camelCase` (`optionalCommonName`, `listingStatusName`) |
| OSM Nominatim | `nominatim.openstreetmap.org/reverse` | Resolved place name, so a bad fix announces itself | — |

**Casing chaos is real.** Three services, one federal government, three
conventions. Never trust documented field names; read what the server
actually returns and match attribute keys case-insensitively, always.

**The silent failure mode.** These older MapServers can ignore
`outFields=*` and return features carrying *only* an OBJECTID — no error,
no warning, just hollow features. Detect it (features present but ≤1
attribute key) and retry with explicitly named fields. Government servers
fail silent, not loud. Design for it.

---

## 2. The ratio that decides the architecture

Of four real places tested, exactly **one** (Beef Basin Road) was answerable
by federal activity data alone. The other three — the region's flagship
bike park (BLM), a university research forest (OSU), and the most famous
trail in the Northwest (NPS) — were invisible to the federal activity
layers and answerable only by *ground layer + hand-maintained rules table*.

So: **a verdict is a lookup crossed with a table.** The lookups are live
(where you are, what data covers that spot). The table is small,
hand-verified, dated, and yours. The table is not a workaround for missing
data; for most land it is the correct representation of how the rules
actually work.

## 3. Verdict order (the algorithm)

1. **Hard designation no's** from PAD-US at the point — Wilderness kills
   bike and vehicle verdicts, NPS kills collection — checked before
   anything else, so no data quirk can override them.
2. **Live activity data** — trail bike fields, MVUM designations — wins
   over generic table rows when present.
3. **Rules table row** for the land manager, most specific match first
   (named unit beats agency beats fall-through).
4. **Honest unknown**: "we don't have a verified answer here" with the
   manager's contact. This is a designed screen, not an error state.

Every verdict carries: manager name, confidence tier, plain-language
confidence sentence, source link, verified-on date, closures link where
known, and the ground-truth closer.

---

## 4. Gotchas earned in the field

**Geometry over names, absolutely.** A nationwide name search for "Sandy
Ridge" found a trail with that exact name — in `admin_org 080804`, the
*Southern Region*, an East Coast forest. Region codes: 06 = Pacific
Northwest, 08 = Southern. Verdicts come from point/radius geometry, never
from name matches. Name search is a diagnostic tool only.

**Trail attribute tiers.** Forests publish one of three subsets:
centerline-only, basic, or management (`attributesubset: TrailNFS_MGMT`).
Only the management tier carries bike fields. A NULL restriction means *no
recorded restriction*, never "allowed" — trails signed hiker-only on the
ground routinely show NULL in the data. Hedge every data-driven "yes";
signs on the ground outrank the database.

**MVUM absence is the one meaningful absence.** Since the 2005 Travel
Management Rule, no designated route = motor travel not authorized on NFS
land. This is the single dataset where an empty result supports a
confident "no" — with one hedge: jurisdiction seams. Beef Basin returned
2.35 miles of FS 093; the rest of the drivable road is county CR 104,
outside the MVUM entirely. State, county, and private timber roads thread
through every forest boundary. `jurisdiction: FS` on the returned feature
is the check.

**Legal ≠ drivable.** Beef Basin: designated open to *all* vehicles,
yearlong — and `operationalmaintlevel: 2 — HIGH CLEARANCE`. Surface both.

**The closure gap.** Fire closures, washouts, and gates live in forest
orders, never in MVUM or trail data. Both Oregon forests carried live road
and wildfire closures during testing that no API exposed. The alert pages
follow a consistent pattern — `fs.usda.gov/r06/{forest}/alerts` — so every
Forest Service verdict links the right forest's alerts page, always,
unconditionally. Link the ground truth; never claim to know it.

**The parking-lot problem.** Trailheads, gates, and pullouts routinely sit
on private or unlisted parcels while the trails are on public ground a few
hundred meters away. Users tap from exactly those spots. Fix: a distance
cascade — exact point, then 150 m, then 500 m — with **labeled honesty**:
when a wider ring matches, the verdict says so and states the distance.
Never silently fuzz the location.

**A wrong coordinate is invisible.** A preset pin for Sandy Ridge was 11 km
off, and three rounds of testing produced confident, well-formatted,
completely wrong answers with nothing on screen to suggest anything was
amiss. Every honesty mechanism — confidence tiers, dated rows, hedged
wording — assumes the location is right. Verify coordinates against a
geocoder before trusting any result, and show the resolved place name back
to the user so a bad fix announces itself. Retested at the true trailhead
(45.3796, -122.0300), the finding held: no Forest Service trail within
250 m, verdict from the table, BLM ground inside the Sandy River ACEC.

**Named recreation units sit beside the parcel you tap.** At Sandy Ridge
the point returns the generic BLM parcel while "Sandy Ridge Trail System"
appears only in the 2 km ring — the trailhead lot is not the trail system.
Nearby-ground logic should eventually prefer a named recreation unit over
a generic parcel.

**Two radii, one query.** Point-in-polygon answers "whose rules govern
here." The 2 km ring answers "what's adjacent" — which is precisely the
soul-card and nearby-ground feed (wilderness boundaries ahead, ACECs,
neighboring parks). The verdict and the soul cards are the same query at
two radii.

**Filter nearby suggestions by access.** The near-miss pointer once
recommended "public ground close by" that was a *privately-owned
conservancy parcel, closed to the public* (`Pub_Access: XA`). PAD-US
access codes: `OA` open, `RA` restricted, `XA` closed, `UK` unknown. Skip
XA and anything described as privately-owned; prefer federal/state
managers. Otherwise the app politely directs people onto land they can't
enter.

**Access was free the whole time.** PAD-US carries `Pub_Access` on every
record — `OA` open, `RA` restricted, `XA` closed, `UK` unknown — so "can I
be here" needed no new source, just reading a field already in hand.
Prefer the ownership (Fee) record's code: designations stacked on top
carry their own, often stricter, values (Wilderness reads `RA`). Surface
the raw code with its caveat; these are coarse and can be years old.

**City and county parks are a real coverage tier.** The first GPS test
from a city park fell straight through to honest-unknown, because the
table had federal and state rows and nothing local. Local parks are the
most likely ground for a casual user to be standing on.

**PAD-US wrinkles.**
- Overlapping records stack: a point returns Fee ownership *plus* every
  Designation on top (park + wilderness + roadless; BLM + ACEC + Wild &
  Scenic). Merge them — never pick one and discard the rest. The stack is
  the answer.
- State-sourced parcels can be a decade stale (`Src_Date` 2012–2013 on
  Oregon rows vs 2021 federal) and contain outright errors (an OSU parcel
  recorded as "University of Oregon"). PAD-US is the best ground layer
  that exists, and it still needs hedged copy.
- Cairn's rule stands: empty result = "likely private or unlisted," with
  *likely* doing load-bearing work.

**Per-forest policy is a phone call, not a page.** Agency websites are not
a reliable rule source: the Forest Service's 2025–26 site migration removed
policy pages outright, and neighbouring forests under the same agency
publish contradictory guidance. District discretion is the system design,
not a documentation failure — which is exactly why the hand-built table is
its correct mirror, and why any activity whose rules live only on those
pages needs verification before it ships.

---

## 4b. The species layer (IPaC)

The soul feature's data source, and the one pipe that behaves differently
from the rest.

**It works from the browser — no proxy, no key.** IPaC takes a `POST` with
a GeoJSON polygon footprint (we send a ~150 m box around the pin), not a
GET point query. Federal POST + CORS from a phone was the open question;
the answer is yes. Everything else in Varda's architecture holds.

**Range, not sightings.** IPaC returns species whose *known or expected
range* covers the footprint. It's a planning tool, and the copy has to say
so: "may range here," never "is here." Absence isn't proof either. This is
the same hedge as "likely private," applied to living things.

**Sort the legal tiers or the feature lies.** One response mixes ESA
listings (`Endangered`, `Threatened`, and their `Proposed`/`Candidate`
variants) with species of concern, birds of conservation concern, and
`Not listed`. Only the first tier carries legal force. Lumping them
inflated a real count of 10 into a headline of 25 — the exact false
precision the honesty rule forbids. Split them, label the second tier
plainly as watched-but-not-protected.

**Read what the service sends, not what its keys promise.** The response
carries a `migbirds` bucket. At every Oregon location tested it came back
**empty** — while the birds themselves arrived inside the populations list
tagged with a bird `groupName` and a "Species of concern" status. Derive
the bird list from the populations, not from the bucket named after it.
(Same lesson as the casing chaos, second verse.)

**Ignore internal bookkeeping values.** `Resolved taxon` is a taxonomy
artifact, not a legal status. Displaying it on the Bald Eagle invited
exactly the wrong inference. Unrecognised status codes are hidden rather
than shown.

**The eagle rule is the best card in the app.** Bald eagles were delisted
in 2007, so they show up outside the ESA tier — but they remain protected
under the Bald and Golden Eagle Protection Act, and a found feather is
still a federal offence to keep. People break this one constantly and in
complete good faith, because the news said "recovered." That's the soul
feature's whole thesis in one card: the rule you didn't know existed.

**Three buckets don't work, and we stopped guessing.** `crithabs`,
`refuges`, and `wetlands` returned 0 everywhere — including a query taken
standing *inside* William L. Finley National Wildlife Refuge, which holds
both a refuge and designated critical habitat. That's conclusive: it's a
request gap, not absence. Rather than keep guessing at parameters, refuge
detection moved to PAD-US, which already carried the refuge name in the
manager field. Ask the source that answers. Critical habitat remains
genuinely unavailable to us for now — noted as a known hole, not quietly
skipped.

**Refuges are their own coverage tier.** The National Wildlife Refuge
System is the federal land anglers, hunters, and field biologists touch
most, and the table had no rows for it until Finley exposed the gap. Rules
worth knowing: designated areas and seasons only, wildlife first, hunting
in set units during set seasons, drones prohibited system-wide.

## 5. The rules table

One row = activity × land match → verdict, one-line why, confidence tier,
source name + link, verified-on date, optional contact, optional closures
link, and a hand-verified flag. Ordering is the logic: hard no's first,
named units before agencies, fall-throughs last; first match wins; live
data slots in between tiers 1 and 3.

**Staleness is enforced by code, not diligence.** Every row carries
verified-on and review-by dates; confidence *degrades automatically* as
rows age — verified voice → hedged voice → "contact this office" — with no
human in the loop. The table rots toward honesty, never toward false
confidence. Roughly twenty lines of date arithmetic; the difference
between a liability and a product.

**Rows have half-lives.** CFR-cited rows (the Wilderness Act, 36 CFR
261.13's travel rule) are near-permanent. Forest-policy-page rows: annual
review.
Phone-call rows are the most expensive kind — log who said what and when,
and try to upgrade them to documented sources. Closures never get rows at
all; they stay live links, permanently outside the rot problem.

**Coverage follows revenue.** A region ships only when seats sold there
fund its upkeep. An honest coverage map beats thin national coverage.

**Verdicts are records.** Store where, when, who, activity, answer,
sources, confidence — and *which table version produced it*. "What did the
app tell our crew on June 3rd" must be answerable with what the table said
then. (Full teams-structure notes live in the project conversation:
org-ownership from user one, roles in the schema, off-the-shelf auth.)

## 6. Soul cards

Two kinds in one table: `legal` (orange — what you didn't know to ask) and
`steward` (pine — how a good guest behaves). One stewardship card per
verdict, most specific match wins; at most two legal cards, data-generated
ones first (seasonal dates, high-clearance warnings pulled live from MVUM
attributes outrank static cards).

The voice rule for every card ever written: **describe the good guest,
never the bad one.** "The best collectors leave a site looking unvisited"
lands; "don't trash the site" scolds. Stewardship rows age slowly and sit
outside the review treadmill; credit Leave No Trace and kin like any
source. Signature cards proven in testing: the e-bike split (motor vehicle
on federal land, bicycle in Oregon State Parks), the wilderness-boundary
warning, ACEC light-touch, the live-data pair (seasonal dates,
high-clearance), the public-land-private-way-in card, and the two species
cards — the feather rule with its live bird list, and the eagle card
(delisted in 2007, still protected, still a federal offence to pocket a
feather). The collection-layer cards — ARPA artifacts and vertebrate
fossils, mining claims, MBTA feathers — are written and parked in section 8.

## 7. Standing sentences

Carried from Cairn, extended for Varda; they appear wherever sourced
answers appear:

> Land status from the USGS Protected Areas Database. Species, birds, and
> habitat from the US Fish and Wildlife Service IPaC. Place names from the
> OpenStreetMap community. Trail and road data from the US Forest Service
> Enterprise Data Warehouse. Rule summaries are hand-maintained and dated;
> each links its source. Varda informs; it never claims authority over the
> ground. Always confirm rules and conditions locally — the land's own
> stewards know it best.

Name every source. Date every rule. End at the ground.

---

## 8. Parked: the collection layer

Researched, verified where possible, cut from v1. Restoring it means
re-adding rows, not redoing research.

- **BLM casual collection** is the one solidly documented rule: personal,
  non-commercial collecting up to 25 lb/day plus one piece, 250 lb/yr,
  hand tools only, not in developed sites — 43 CFR 3622. Near-permanent row.
- **NPS is a blanket no** (36 CFR 2.1); wilderness effectively likewise.
- **Forest Service is per-forest.** The Gifford Pinchot requires a permit
  for *all* rockhounding including small personal amounts; the Willamette's
  policy page no longer exists; the Deschutes mostly points at neighbouring
  counties. Willamette Supervisor's Office 541-225-6300, Deschutes
  541-383-5300 — the calls that would turn two honest-unknowns into dated
  rows, whenever collection returns to scope.
- **Mining claims** can't be checked from any free spatial source; the
  honest treatment is a card telling people to watch for claim posts and
  ask the field office, never a verdict.
- **Soul cards already written:** ARPA — arrowheads, pottery, anything
  made by human hands, plus vertebrate fossils needing a scientific permit;
  MBTA — most wild bird feathers are illegal to keep, even found ones;
  a mining-claim card; and the stewardship line, *take what you'll truly
  treasure, not what you can carry; the best collectors leave a site
  looking unvisited.*
