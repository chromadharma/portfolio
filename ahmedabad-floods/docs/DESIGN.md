# DESIGN — Flooding Ahmedabad: who gets cut off?

**Repo:** `ahmedabad-floods` (proposed; drafted inside `chromadharma/portfolio`
until you confirm the name, see D1) · **Engine:** `hazardnet`, imported from
`chromadharma/ca-road-fragility` · **Author:** Sahasrik Ragani
**Status:** draft for review, 26 Sep 2026. Nothing downloaded or built. The only
things fetched were Sentinel-1 scene *metadata* (footprint JSON, a few kB each),
the first 1 kB of two DEM tiles, and the DataMeet ward file (1.2 MB) for
inspection. None of it is committed.

---

## 1. Question

**Can a flood model for Ahmedabad be built that is measurably more accurate
than the tools used now, and if so, which parts of the city lose road access
to a hospital, and at what level of flooding?**

The first half is the project. The second half is what the model is for.

### The case that sets the bar: 23–25 July 2026

On 23 July 2026, AMC's gauges recorded 11.20 inches (284 mm) at Bakrol,
9.65 in at Bopal, and 9.49 in at Sarkhej Urban Health Centre in about twelve hours.
The South West Zone averaged 196 mm, the North West 141 mm and the West 110 mm
([DeshGujarat, 23 Jul 2026](https://deshgujarat.com/2026/07/23/ahmedabad-city-records-over-11-inches-of-rain-in-12-hours-area-wise-rainfall-data-here/),
[ward-wise table](https://deshgujarat.com/2026/07/23/where-did-it-rain-in-ahmedabad-city-ward-wise-rainfall-data-here/)).
Water stood about three feet deep in Bopal and Ghuma for nearly three days. By
5 pm on 25 July AMC had cleared 107 of 126 waterlogged housing societies
([DeshGujarat, 25 Jul 2026](https://deshgujarat.com/2026/07/25/rainwater-cleared-from-107-of-126-waterlogged-societies-in-ahmedabad-amc/)).
At the same time, the Vasna barrage stood at 134–135 ft against a normal
operating level of about 128 ft (Counterview, Aug 2026, via search snippet only;
the page is blocked from here).

**Sentinel-1D passed over the city at 01:09 UTC (06:39 IST) on 25 July.** Its
swath edge cut the city at 72.546° E. It imaged the western strip
(Gota, Chandlodia, Thaltej, Bodakdev, Jodhpur, Vejalpur, Sarkhej, Maktampura,
plus Bopal–Ghuma–Shela) and stopped roughly 2–3 km short of the Sabarmati. I
checked this against the scene footprint (§4). By chance, this one pass covered
the worst-hit belt while the water was still standing.

This event is why "a better model" means something specific here. The brief's
first-pass method, HAND (height above nearest drainage), measures how far a
cell sits above the nearest channel. It is built for **rivers spilling over
their banks**. Bopal's July 2026 flood was **rain ponding in low ground and
overwhelming drains**, about 10 km west of the river. My expectation, to be
tested and not assumed: HAND will largely miss Bopal, and a model that routes
rain into depressions will catch it. If that expectation fails, it goes in
`FINDINGS.md` with the numbers.

### What "better" is measured against

"More accurate" is only a claim if it names what it beats and on what data.
The model ladder (§5.2) is scored on the same observations, on the same grid,
with the same metrics:

| Rung | Model | Why it's on the ladder |
|---|---|---|
| M0 | **Null:** flood the lowest *x*% of cells, with *x* set to the observed flooded share | Any model has to beat this, or its skill is just "low ground floods" |
| M1 | **HAND + Sabarmati bathtub** (the brief's first pass) | The standard fast, low-data method, and the one most studies use |
| M2 | **Fill–spill (pluvial):** rain minus drain loss fills depressions and spills downhill | The cheapest model that can represent waterlogging at all |
| M3 | **2D rain-on-grid hydrodynamic** (LISFLOOD-FP 8): rain plus a river boundary, over time | The physically fuller model; the "better" candidate |

There is one published Ahmedabad 2D model (HEC-RAS 5.0.3 on AW3D30, riverine
only, for the 2006 flood;
[ResearchGate 324538280](https://www.researchgate.net/publication/324538280_Application_of_2D_HEC-RAS_Hydrodynamic_Modelling_for_Flood_Inundation_Mapping_-_A_Case_of_Ahmedabad_City_Gujarat_India)).
It is context, not a baseline: its inputs and outputs aren't public, and 2006
predates Sentinel-1. I found no public Ahmedabad model that handles rain-driven
flooding and is validated against satellite observations. That gap is the
contribution, but I have only searched the open web so far (D10).

---

## 2. How the two halves connect

```
DEM(s) ─┬─ M1 HAND/bathtub ─┐
rain ───┼─ M2 fill–spill ───┼─► depth rasters per scenario ─► hazardnet ─► time to nearest hospital
river ──┴─ M3 LISFLOOD-FP ──┘          │                     (roads)        per hex / ward, per scenario
                                        └─► validation vs Sentinel-1 + reported locations
```

The network step runs on whichever rung validates best. It also runs on M1,
so the write-up can show how much the choice of flood model changes who is
counted as cut off. If the answer is "a lot", that is a finding in its own
right.

---

## 3. Data sources: what was verified, and how

This cloud environment blocks most of the hosts this project needs. Blocked:
Overpass, IMD, Bhuvan/NRSC, Planetary Computer, Copernicus Data Space,
Geofabrik, data.bris (FABDEM), AMC, and all news sites. Open: AWS S3 and GitHub.
So "verified" below means one of three things:
**✅ pulled from here**, **🔎 confirmed by web search only** (reachable per
its documentation, not yet pulled by a script), or **❓ unverified**.

| Layer | Source | Status | Licence / terms | Size, signup |
|---|---|---|---|---|
| DEM (primary) | Copernicus GLO-30 COG, AWS `copernicus-dem-30m`, tiles N22E072 + N23E072 | ✅ range-read both tiles | Copernicus DEM licence (free, attribution); exact wording to confirm for `DATA_LICENSES.md` | 45.5 + 44.6 MB, no signup |
| DEM (bare-earth) | FABDEM v1-2, Univ. of Bristol (GLO-30 with buildings and trees removed) | 🔎 | **CC BY-NC-SA 4.0**: fine for a portfolio, but ShareAlike applies to any depth raster derived from it that we publish | ~tens of MB, no signup |
| DEM (national) | CartoDEM v3 30 m, ISRO Bhuvan | 🔎 | **Bhuvan terms forbid derivative works and redistribution.** Private users may need ISRO/DOS clearance | Bhuvan login required (D3) |
| Observed extents | Sentinel-1 IW GRD, AWS `sentinel-s1-l1c` | ✅ anonymous read of metadata *and* image TIFFs (HTTP 206) | Copernicus Sentinel data, free and open; credit "contains modified Copernicus Sentinel data [year]" | **~1.26 GB per scene** (VV 692 MB + VH 571 MB) (D4) |
| — alternative | Google Earth Engine S1 GRD | 🔎 | Needs a Google Cloud project registered for non-commercial use; 2026 quota tiers | Signup (D4); not needed |
| — alternative | Planetary Computer `sentinel-1-rtc` (terrain-corrected COGs) | ❓ blocked here; S1D coverage in 2026 unknown | CC BY 4.0 per PC docs (to confirm) | No account for reads |
| Rain (gridded) | IMD 0.25° daily (`imdlib`) | 🔎 archive runs **1901–2024**; 2025/2026 not yet in the archive product | IMD terms (to confirm) | No signup. **0.25° ≈ 27 km covers the whole city in 2–4 cells**, far too coarse for the Bopal-vs-east contrast |
| Rain (gauges) | AMC ward and zone gauge totals, as reported in the press for 23 & 25 Jul 2026 | 🔎 secondary (transcribed news) | News reports, cited by URL | We need AMC's primary table (D5) |
| Rain (satellite) | GPM IMERG half-hourly, 0.1° | ❓ | NASA open data | NASA Earthdata login (D5) |
| River | Vasna barrage level and releases: 1.85 lakh cusecs peak (2017), 90,000 (2019), 50,000 (2022), >1.07 lakh (Sep 2025), 134–135 ft (Jul 2026) | 🔎 news via search | — | Needs an official series if one exists (D5) |
| Roads, hospitals, waterways, underpasses | OpenStreetMap via Project 1's loaders (Overpass, or Geofabrik PBF fallback) | ❓ both blocked here; hospital count unknown | ODbL 1.0 | PBF size to check (<1 GB expected) |
| Admin units | DataMeet `Municipal_Spatial_Data/Ahmedabad/Wards.geojson`, 48 wards | ✅ | CC BY-SA 2.5 IN; scraped from an AMC Google map | 1.2 MB. **Does not contain Bopal or Ghuma** (point-in-polygon test), so vintage is probably pre-2020 expansion |
| Buildings, land cover, population | Google Open Buildings / ESA WorldCover / WorldPop or GHS-POP | ❓ | CC BY 4.0 (to confirm each) | Optional; M3 and the "who" layer |

**Implication of the ward gap.** Bopal–Ghuma–Shela is exactly where the 2026
flood and the 2026 SAR overlap, and the only usable ward file leaves it out.
So the **primary unit is a hex grid** over the model domain (proposed: H3
resolution 8, ~0.74 km² per cell, fine enough to separate one side of SG Highway
from the other). Wards are a second aggregation wherever they cover. If you
have or can get AMC's current 48-ward layer, it replaces DataMeet (D6).

---

## 4. Flood events to validate against

I listed every Sentinel-1 IW scene whose footprint touches the AMC extent
(72.45–72.70° E, 22.91–23.14° N), day by day, around each candidate event,
using the AWS archive's per-scene `productInfo.json`. Two tracks cover the city:
**track A** (≈01:02 UTC) covers all of it, and **track B** (≈01:10 UTC) covers
only the strip west of ≈72.55° E. Since S1B failed in Dec 2021, each
track repeats every 12 days, now flown by S1A or S1D.

| Event | What is documented (source) | S1 scenes over the city | Verdict |
|---|---|---|---|
| **E1: 23–25 Jul 2026, pluvial** | 284 mm Bakrol, 245 mm Bopal in ~12 h; 3 ft standing in Bopal–Ghuma ~3 days; 126 societies waterlogged; Vasna at 134–135 ft (DeshGujarat; Counterview) | A: 20 Jul (pre) · **B: 25 Jul 01:09 UTC, western strip, during** · A: 1 Aug (post) | **Primary test event.** West only; ~2 days after peak rain |
| E2: ~7 Sep 2025, fluvial + pluvial | >1 lakh cusecs in the Sabarmati, lower promenade under water, ~300 waterlogging complaints, most from the East Zone (DeshGujarat, 7 Sep 2025) | A: 4 Sep · B: 9 Sep (strip excludes the river) · A: 16 Sep | River extent likely **not observed near peak**; confirm peak date |
| E3: ~24–26 Aug 2025, fluvial | ~51,000 cusecs through 25 gates; Dholka, Vatva, Vejalpur, Bhat, Daskroi flooded (ETV Bharat) | A: 23 Aug · B: 28 Aug | Weak; mostly downstream of the city |
| E4: late Jul 2017, fluvial | Vasna peak 1.85 lakh cusecs, Dharoi release 1.3 lakh; 18 rain deaths in Ahmedabad (Wikipedia, via search) | A: 24 Jul · B: 29 Jul (x ≤ 72.552) · A: 5 Aug | Best riverine candidate **only if** 24 Jul is near peak; peak date unverified |
| 2019, 2022, 22 Jul 2023, Jul 2023 river raise for U20 | Discharges and dates from search snippets only | not yet listed | **Your knowledge wanted (D7)** |

Two honest consequences follow.

1. **Riverine validation from SAR is thin.** No listed pass catches the river at
   a verified peak. So the river component gets a second, non-SAR test. The
   lower promenade is a structure at a known elevation, and it has a documented
   record: underwater at >1 lakh cusecs (2025), and closed at ~54,000 cusecs in
   another year whose date we still need. Given each reported discharge, M1 and
   M3 must submerge it or not. It is a crude test, but a real one.
2. **Pluvial validation rests on one event and one strip.** Any accuracy claim
   is about E1's western belt and says so. To avoid tuning on the test data,
   calibration and testing are split within E1 by space: calibrate on the
   northern half of the strip, test on the southern half, then swap and report
   both. If a second observed pluvial event turns up (D7), that becomes the
   test set instead.

---

## 5. Method

### 5.1 Grid and DEM

- Work in UTM 43N (EPSG:32643) on a 30 m grid. Domain: the AMC limits including
  Bopal–Ghuma, plus a buffer for upstream catchment. The domain polygon source
  is still open; OSM admin boundaries are the first candidate.
- **Three DEMs, one comparison.** GLO-30 is a surface model: it includes
  buildings, which is wrong for water on streets. FABDEM removes buildings and
  trees. CartoDEM is the national product. Every rung runs on GLO-30 and
  FABDEM; CartoDEM runs only if D3 clears, and nothing derived from it is
  published. The DEM effect is reported as a finding, not tuned away.
- **Two conditionings, deliberately different.** HAND needs a hydrologically
  "clean" DEM, with depressions breached so that every cell drains. The pluvial
  rungs must *keep* the depressions, because a depression is where waterlogging
  happens. Filling them would delete the signal we're trying to model.

### 5.2 The model ladder

**M0, the null.** Rank cells by elevation. Flood the lowest *x*%, with *x* set
to the observed flooded share in the scored area. It has no free parameters.

**M1, HAND plus bathtub (the brief's first pass).** Compute HAND with
WhiteboxTools, or `pysheds` as a fallback. Drainage lines come from a
flow-accumulation threshold, checked against OSM `waterway=*`. Depth scenarios:
HAND ≤ *h* for *h* ∈ {0.25, 0.5, 1, 1.5, 2, 3} m. Along the Sabarmati, a
bathtub fill at Vasna barrage stage levels {128, 131, 134.75, 138} ft, plus
the reported 2017 peak once its stage is found. It is stated plainly in the
README that HAND represents river overbank flow and does not represent rain
ponding away from channels.

**M2, fill–spill.** Take rainfall minus a drainage and infiltration loss (mm/h,
one calibrated parameter). Fill each depression to capacity, then spill the
excess to the next depression downhill (Barnes, Callaghan & Wickert 2020,
Fill–Spill–Merge; `richdem`, GPL-3). It is static, so it gives maximum
depths, not timing. Scenarios: 24 h rain totals {50, 100, 150, 200, 250,
300} mm, spread in space using the AMC gauge pattern where available.
`richdem` build status on Apple Silicon is unverified; a small NumPy
implementation is the fallback.

**M3, LISFLOOD-FP 8 rain-on-grid.** A 2D local-inertial shallow-water model
(Bates, Horritt & Fewtrell 2010) with spatially and temporally varying
rainfall, a river inflow and stage boundary, and a drain-loss sink. Why
LISFLOOD-FP and not HEC-RAS:

- LISFLOOD-FP 8.x is open source (GPL-3). It builds with CMake on Linux and
  Windows. A community QGIS plugin reports it working on macOS Apple Silicon.
  Rain-on-grid has been supported since 8.1
  ([GMD 2023](https://gmd.copernicus.org/articles/16/2391/2023/)).
- HEC-RAS 6.x is Windows-only, apart from a Linux command-line solver.
  **HEC-RAS 2025** is a C#/.NET rewrite said to run natively on Linux, but as of
  June 2026 it is a public beta "not recommended for production", and I found
  no evidence of macOS support
  ([Civinnovate, Jun 2026](https://civinnovate.com/2026/06/17/hec-ras-2025-complete-guide/),
  a secondary source). On a Mac it would mean a VM or CrossOver. Recommendation:
  **LISFLOOD-FP as the M3 engine, HEC-RAS not used.** Revisit only if a reviewer
  asks for it, or if HEC-RAS 2025 ships a stable Linux or Mac build.

At 30 m, the domain is roughly 1 million cells, and an event of a few days is a
CPU job measured in hours on a laptop (to be timed). Calibration covers two
parameters, drain loss and a single Manning's *n*, and is done on the
calibration half of E1 only.

**What no rung can see: underpasses.** Ahmedabad's cut-offs often happen in
underpasses, such as Mithakhali, Akhbarnagar, Makarba, Dakshini and Shahibaug,
all closed in recent floods. These are narrower than a 30 m cell and don't
exist in any 30 m DEM. They are handled on the network side instead. OSM
`highway=*` ways with `tunnel=yes` or `layer<0` are flagged, and a scenario
closes them at a low rainfall threshold regardless of modelled depth. The
README reports results both with and without that rule. How completely OSM
tags these underpasses is unverified.

### 5.3 Observed flood extent from Sentinel-1

- **Pre/during change detection.** For open ground, flooding lowers VV and VH
  backscatter; the threshold comes from Otsu on the log-ratio. For built-up
  areas, flooding can *raise* backscatter through double bounce between water
  and walls. That signal is noisier and is marked experimental.
- **An "observable" mask.** Cells in radar layover or shadow, permanent water,
  and dense built-up cells where neither signal is reliable are **excluded from
  scoring**. The remainder is mapped and its share reported. A flood the
  satellite cannot see is not counted as a model miss.
- **Timing.** The E1 image is from about 36–48 h after peak rain. M3 is scored
  at the matching model hour; M1 and M2 give maximum extents only and will
  over-predict for that reason. The README says so beside the numbers.
- **Processing without SNAP.** Calibration and terrain correction use
  `sarsen` (pure Python; licence to confirm), reading only the city window from the AWS
  TIFFs where possible. If that proves unreliable, we fall back to Planetary
  Computer RTC tiles or Earth Engine (D4).

### 5.4 Validation metrics

Scores are computed on the observable mask, on the 30 m grid, with SAR water
aggregated from 10 m.

- **Hit rate** H = hits / (hits + misses).
- **False alarm ratio** FAR = false alarms / (hits + false alarms). The brief
  says "false-alarm rate"; that name also refers to a different quantity,
  false alarms / all dry cells. Both are reported, labelled, so neither is
  mistaken for the other.
- **Critical success index** CSI = hits / (hits + misses + false alarms).
- **Skill over the null:** CSI(model) − CSI(M0). Raw CSI rises with the size of
  the flood (Stephens, Schumann & Bates 2014), so a model is "better" only if it
  beats the null on the same scene.
- **Points.** Geocoded reported waterlogging locations (press reports and AMC
  lists, if we get them) give a hit rate only; they can't show false alarms.
  They are also **biased toward areas the press covers**. In 2025 the East Zone
  logged the most complaints (64), while coverage of 2026 concentrated on the
  western belt. That bias is written up, not corrected for silently.

**Pre-registered claim (for you to set, D9).** Before anything is run, we
write down what "better" means. A proposal: M3 (or M2) beats M1 by ≥ 0.10 in
CSI skill over the null on the test half of E1, on both DEMs. If it doesn't,
the project reports that, and the finding becomes what it would take
(drainage data, a better DEM) to close the gap.

### 5.5 Network step (via `hazardnet`)

For each scenario on each rung:

1. Load the depth raster with `load_hazard(kind="raster", bins=[0.15, 0.30, 0.50])`,
   then `overlay()`. Each road edge gets `hazard_max` (1: 15–30 cm, 2: 30–50 cm,
   3: ≥50 cm) and the share of its length in each class.
2. **Removal and slowing.** Edges with any length at ≥ 30 cm are removed, since
   car speed falls to roughly zero near 0.3 m (Pregnolato et al. 2017,
   depth–disruption function). Edges at 15–30 cm keep `travel_s` inflated by
   the same function. That adjustment is done study-side, by rewriting the edge
   attribute, and needs no engine change. Underpass rule as in §5.2.
3. **Targets.** Hospital access nodes come from OSM `amenity=hospital`. A
   hospital whose own access node is flooded is dropped as a target. OSM's
   hospital coverage in Ahmedabad is unknown, and we'll cross-check it against
   an official list (D8).
4. **Measure.** `time_to_targets()` gives seconds from every node to the
   nearest reachable hospital, in one Dijkstra run per scenario. Per hex:
   the median node time, plus a flag if no node reaches any hospital.
   `accessibility()`'s max-flow is not needed for the core question (it answers
   "how many ways out", not "how far to care") and is kept for an optional ward
   comparison.
5. **Cut-off level.** For each hex, the lowest scenario level at which it is
   cut off. There are two readings of "at what depth", and both are reported:
   (a) the **forcing** level, such as "cut off at 150 mm of rain in 24 h" or
   "at Vasna 134.75 ft"; (b) the **local depth** on the road that binds, found
   by walking back from the removed edges. (a) is what a planner can act on;
   (b) is the literal question. See D9.

---

## 6. Engine interface

**Project 1 is ready.** `chromadharma/ca-road-fragility` (main, 26 Sep 2026)
ships `hazardnet/` with tests. Its `Hazard` class already anticipates "a raster
with CONTINUOUS values (flood depth, for Project 2)". Its public API, which
this project uses unchanged:

```python
from hazardnet import load_roads, load_hazard, overlay, time_to_targets, accessibility
from hazardnet.osm import configure

configure(cache_dir, logs_dir)                        # Overpass hygiene from Project 1
rg  = load_roads(domain_polygon_4326, source="osm",
                 capacity_table=CAPACITY)             # RoadGraph (directed igraph + edge GeoDataFrame)
for sc in scenarios:                                  # one depth GeoTIFF per scenario
    hz = load_hazard(sc.depth_tif, kind="raster",
                     levels={1: "15-30cm", 2: "30-50cm", 3: ">=50cm"}, bins=[0.15, 0.30, 0.50])
    overlay(rg, hz, metric_crs="EPSG:32643")          # writes hazard_max, hazard_share_k
    g = drop_and_slow(rg, sc)                         # study-side: remove >=30cm, slow 15-30cm, close underpasses
    t = time_to_targets(g, hospital_nodes(g, sc))     # seconds to nearest hospital, per node
```

**How it's imported, not copied.** `ca-road-fragility` has no `pyproject.toml`,
so it can't be pip-installed. Proposal: add a minimal one there (about 15 lines,
declaring `hazardnet` and its dependencies), tag it `v0.1.0`, and depend on
it here with:

```
hazardnet @ git+https://github.com/chromadharma/ca-road-fragility@v0.1.0
```

This changes Project 1's repo, so it needs your approval (D2). Until then, a
pinned git submodule is the fallback. It is still not a copy.

**Gaps found reading the code.** None of them block the start.

| # | Gap | Proposal |
|---|---|---|
| G1 | Not installable | `pyproject.toml` upstream (above) |
| G2 | `_overlay_raster` samples each edge in a Python loop and reprojects per edge. That's fine once, but slow for ~100k edges × dozens of scenarios | Time it first. If it's slow, upstream an `overlay_many(rg, rasters)` that computes sample points once |
| G3 | `osm.UA` hard-codes "ca-road-fragility research" as the Overpass user-agent | Upstream a `user_agent=` argument to `configure()` |
| G4 | No depth-to-speed slowing | Study-side: rewrite `travel_s` before `time_to_targets` (the engine reads the attribute by name) |

Nothing Ahmedabad-specific goes into `hazardnet`. Everything specific lives in
`studies/ahmedabad/config.yaml` here, which is the same split Project 1 uses.

---

## 7. Outputs

- **Observed extents:** an E1 Sentinel-1 flood map with its observable mask,
  built with the `imagery-exhibits` skill, since the finding rests on pixels.
  Every panel cites sensor, scene ID, UTC time and pixel size.
- **Validated flood-extent maps** per scenario for M1–M3, each with its
  contingency map (hit, miss, false alarm) for E1.
- **Validation table:** H, FAR, POFD, CSI and skill over the null, per rung ×
  DEM × calibration/test half.
- **Accessibility-loss map** (hex) and a **table of hexes and wards ranked by
  cut-off level**, with population per hex if D8 adds a population layer.
- **Hero figure (proposed):** left, the 25 July 2026 western strip showing
  SAR-observed water against the best rung's prediction; right, the city's
  hexes coloured by the rainfall at which they lose hospital access.
- Charts use `modular-viz-system` (Viridis, Archivo, white #FFFFFF). Every
  figure carries its data source and date.

Repo layout mirrors Project 1: `docs/` (DESIGN, FINDINGS), `data/README.md` +
`scripts/fetch.py`, `studies/ahmedabad/`, `models/` (M0–M3 wrappers and the
LISFLOOD-FP build script), `viz/`, `outputs/`, `tests/`, a `Makefile`
(`make all`), `requirements.txt` + `environment.yml`, `LICENSE` (MIT) and
`DATA_LICENSES.md`.

---

## 8. Risks and limits (these go in the README)

1. **Satellites miss most urban flooding.** C-band SAR sees water in the open.
   Between buildings it sees shadow, layover and double bounce. The observable
   mask may exclude much of the dense core, and the README reports its share.
2. **One pluvial event, one strip, one image ~2 days late.** Any accuracy
   claim is local to E1's western belt. The spatial calibration/test split
   limits over-fitting but doesn't create a second event.
3. **No drainage network.** AMC's storm-water network isn't public, so drains
   are a single loss-rate parameter. This is probably the largest structural
   error in M2 and M3.
4. **30 m can't see streets or underpasses.** That is the reason for the
   underpass rule (§5.2), which is a scenario assumption and not a model output.
5. **Rain forcing is thin.** IMD's gridded product is too coarse and not yet
   published for 2025–26. The gauge totals we have are transcribed from
   news reports.
6. **OSM coverage:** unknown completeness for hospitals, underpass tagging, and
   the lanes and one-way tags that set travel times.
7. **Reporting bias** in point validation (§5.4).
8. **Licence constraints:** FABDEM derivatives are NC-SA. Nothing derived from
   CartoDEM can be published.
9. **Environment:** this cloud session can't reach most sources. `make data`
   has to run on your machine, or the environment's allowlist has to be widened.
10. **Cross-platform:** LISFLOOD-FP is compiled C++, and its Windows and Mac
    builds need testing on your machines. WhiteboxTools, `richdem` and `sarsen`
    are expected to install on Mac and Windows, but that is unconfirmed for
    each package until `make env` runs on both. Nothing chosen is Windows-only.

---

## 9. Decisions needed from you

| # | Decision | My recommendation |
|---|---|---|
| D1 | Repo name and home | New repo `chromadharma/ahmedabad-floods`; I move this doc there once you confirm |
| D2 | Add `pyproject.toml` + tag `v0.1.0` to `ca-road-fragility` so `hazardnet` can be imported | Yes; it's a small change |
| D3 | CartoDEM: register on Bhuvan and use it for comparison only (no published derivatives)? | Yes if you're eligible without ISRO clearance; otherwise drop it and say why |
| D4 | Sentinel-1 download: ~6 scenes × ~1.26 GB ≈ **7.5 GB** full-size, or ~0.5–1 GB with windowed reads. Which processing route? | Approve up to ~8 GB; windowed reads + `sarsen` from AWS; no GEE signup |
| D5 | Rain and river: do you have AMC gauge tables or Vasna barrage records? Sign up for NASA Earthdata (IMERG)? | Ask AMC or the press for the 23 Jul 2026 gauge table; Earthdata signup yes (free) |
| D6 | Unit of analysis: H3 res-8 hexes primary, DataMeet wards secondary? Do you have AMC's current ward layer? | Hexes primary |
| D7 | Flood events you know of: especially the peak dates for 2017, 2019 and 2022, and any second pluvial event with a clean date | — |
| D8 | Hospital list to cross-check OSM, and whether to add population (WorldPop or GHS-POP) | Yes to population; you may know the best official hospital list |
| D9 | Definitions: cut-off = no hospital reachable, or travel time > *T* min? Report by forcing level, local depth, or both? The pre-registered "better" threshold? | Both readings; *T* = 30 min as a second threshold; ≥ 0.10 CSI skill |
| D10 | Literature search before building, to confirm no validated Ahmedabad pluvial model exists | Yes: a short pass with the `reading-repository` skill |

---

## 10. References

**Web sources checked for this draft**, all via search (pages blocked from here
except where noted):
[DeshGujarat, 23 Jul 2026, 11 inches in 12 hours](https://deshgujarat.com/2026/07/23/ahmedabad-city-records-over-11-inches-of-rain-in-12-hours-area-wise-rainfall-data-here/) ·
[DeshGujarat, 23 Jul 2026, ward-wise rainfall](https://deshgujarat.com/2026/07/23/where-did-it-rain-in-ahmedabad-city-ward-wise-rainfall-data-here/) ·
[DeshGujarat, 25 Jul 2026, 107 of 126 societies cleared](https://deshgujarat.com/2026/07/25/rainwater-cleared-from-107-of-126-waterlogged-societies-in-ahmedabad-amc/) ·
[DeshGujarat, 7 Sep 2025, >1 lakh cusecs](https://deshgujarat.com/2025/09/07/over-a-lakh-cusecs-water-in-river-sabarmati-in-ahmedabad-lowe-promenade-under-water-downstream-areas-alerted/) ·
[ETV Bharat, Aug 2025, Sabarmati release](https://www.etvbharat.com/en/!videos/flood-like-situation-in-sabarmati-river-as-dam-water-released-gujarat-enn25082403647) ·
[Counterview, Aug 2026](https://www.counterview.net/2026/08/did-sabarmati-riverfront-make-ahmedabad.html) ·
[SANDRP on X, dating the 2026 floods to 23–26 Jul](https://x.com/Indian_Rivers/status/2084192128977313971) ·
[Gujarat Samachar, AMC waterlogging spots and underpasses](https://english.gujaratsamachar.com/news/ahmedabad/ahmedabad-deluged-over-8-inches-of-rain-exposes-amcs-multi-crore-pre-monsoon-claims-as-posh-belts-submerge-73506814338) ·
[2017 Gujarat flood (Wikipedia)](https://en.wikipedia.org/wiki/2017_Gujarat_flood) ·
[HEC-RAS 2D Ahmedabad study](https://www.researchgate.net/publication/324538280_Application_of_2D_HEC-RAS_Hydrodynamic_Modelling_for_Flood_Inundation_Mapping_-_A_Case_of_Ahmedabad_City_Gujarat_India) ·
[LISFLOOD-FP 8.1, GMD 2023](https://gmd.copernicus.org/articles/16/2391/2023/) ·
[HEC-RAS 2025 guide (Civinnovate)](https://civinnovate.com/2026/06/17/hec-ras-2025-complete-guide/) ·
[FABDEM v1-2](https://data.bris.ac.uk/data/dataset/s5hqmjcdj8yo2ibzi9b4ew3sn) ·
[IMD 0.25° gridded rainfall](https://imdpune.gov.in/cmpg/Griddata/Rainfall_25_Bin.html) ·
[imdlib](https://imdlib.readthedocs.io/en/latest/) ·
[Bhuvan CartoDEM](https://bhuvan-app3.nrsc.gov.in/data/download/index.php?c=s&s=C1&p=cdv2) ·
[Earth Engine non-commercial tiers](https://developers.google.com/earth-engine/guides/noncommercial_tiers) ·
[DataMeet Municipal_Spatial_Data](https://github.com/datameet/Municipal_Spatial_Data/tree/master/Ahmedabad) (✅ cloned).

**Papers cited from memory.** Each must be checked (DOI, pages) before it
appears in the README or `FINDINGS.md`:
Nobre et al. (2011), HAND, *Journal of Hydrology* ·
Bates, Horritt & Fewtrell (2010), inertial shallow-water formulation, *Journal of Hydrology* ·
Hawker et al. (2022), FABDEM, *Environmental Research Letters* ·
Stephens, Schumann & Bates (2014), binary pattern measures for flood model evaluation, *Hydrological Processes* ·
Pregnolato et al. (2017), depth–disruption function, *Transportation Research Part D* ·
Barnes, Callaghan & Wickert (2020), Fill–Spill–Merge, *Earth Surface Dynamics* ·
Mason et al. (2010), urban flood detection with TerraSAR-X, *IEEE TGRS*.
