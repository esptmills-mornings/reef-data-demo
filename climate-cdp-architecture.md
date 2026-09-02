# Climate Data Unification Platform — Architecture v0.1
### A CDP-pattern approach to climate/environmental data, scoped to an MVP

---

## 1. Core analogy, made precise

| Amperity CDP | This platform |
|---|---|
| Customer | **Location entity** (reef tract, watershed, farm parcel, grid cell) |
| Identity resolution (cookie/email/loyalty card → one person) | **Geospatial resolution** (satellite pixel/buoy/station/IoT sensor → one location entity) |
| Customer 360 timeline | **Location timeline** — every variable, every source, one axis of time |
| "Customers who did X then did Y" | **Lagged pattern detection** — "locations where X happened, Y happened N months later" |
| Segments / audiences | **Cohorts of locations** sharing a pattern (e.g. reefs with >1.5°C anomaly exposure) |
| Activation (email, ads) | **Alerts / model triggers / downstream dashboards** |

The one place the analogy breaks: CDP joins are usually *same-time* ("this session, this purchase"). Climate joins are usually *lagged and diffuse* — an ocean anomaly propagates through currents and atmosphere for weeks to months before it shows up as a local effect. **The lag-alignment engine is the one genuinely novel piece of engineering here; everything else is fairly standard ETL.**

**A second break, found in the prototype and more serious than the first:** a single global driver variable (ONI) does not generalize across entities in different ocean basins, even when it's convenient to treat it as if it does. GBR and MAR both bleached in 2017 under ENSO-neutral conditions — tempting to read as "one shared global anomaly" — but their actual satellite thermal-stress peaks landed seven months apart, driven by two unrelated regional mechanisms (local wind/cloud conditions in the Coral Sea; the Atlantic Meridional Mode in the Caribbean, an index unrelated to Pacific ENSO). A platform that stores only ONI as "the" driver per entity, regardless of the entity's ocean basin, will not just miss the real explanation — it will actively manufacture a false one ("both bleached the same year, so it's one global event") that a person skimming a dashboard would readily believe. Basin-appropriate driver indices (AMV/AMM for the Atlantic, PDO for the Pacific, alongside ENSO) need to be part of the source layer from the start, selected per entity by its ocean basin, not applied as one universal global signal.

---

## 2. MVP scope (deliberately narrow)

- **Entity type:** reef tract (well-instrumented, well-documented, short feedback loop between cause and observable effect — good for a first demo)
- **Question:** Does a lagged SST/ENSO anomaly signal predict bleaching onset at specific reef sites better than either signal alone?
- **Locations:** 5–10 named reef sites with existing NOAA monitoring history (not global coverage — deliberately small)
- **Sources (3, not 30):**
  1. **NOAA Coral Reef Watch** — reef-specific SST anomaly + bleaching alert levels, already gridded to reef sites
  2. **ONI / ENSO index** — NOAA's monthly Oceanic Niño Index, simple CSV, no resampling needed
  3. **NOAA bleaching observation records** — ground-truth outcome variable

This scope proves the full loop — unify → lag-align → detect pattern → visualize — without yet touching raw satellite imagery, IoT ingestion, or global scale. Those are phase 2+.

---

## 3. Entity model (the "customer profile" equivalent)

Decide this **before** any data pipeline work — every source has to crosswalk to it.

```
LocationEntity
├── entity_id            (stable, e.g. NOAA reef site ID)
├── entity_type           (reef_tract | watershed | grid_cell | farm_parcel ...)
├── geometry               (point or polygon, with a defined resolution)
├── ocean_basin            (Pacific | Atlantic | Indian ... — determines which teleconnection indices are even relevant)
├── crosswalk_keys         (per-source IDs: NOAA station ID, satellite pixel index, etc.)
└── timeline[]
      ├── timestamp
      ├── source
      ├── variable          (sst_anomaly, chlorophyll, oni_index, bleaching_alert_level, cyclone_category, coral_cover_pct, fish_biomass_index, ...)
      ├── variable_class     (stressor | response — see note below)
      ├── stressor_type      (thermal | mechanical | chemical | biological ...) — only set when variable_class = stressor
      ├── value
      └── confidence / provenance
```

Key design decisions to make explicit early:
- **Grain**: point vs. polygon. Reefs are naturally polygons; SST grids are pixels. You'll resample one to the other — decide which direction and document the loss. **This isn't just a spatial issue — it distorts apparent lag.** The prototype pulled two real GBR coral trout metrics for the same policy change (the 2004 no-take rezoning): a zone-level comparison (protected vs. fished reefs) showed a measurable effect within ~2 years, while the whole-of-fishery stock assessment — diluted by the two-thirds of reefs that stayed open, plus 50 years of prior fishing pressure baked into the baseline — didn't trough and turn around until 2011. Naively reading the aggregate number alone would wrongly conclude the intervention took 7 years to work. An entity model that only stores one grain per variable will give a wrong answer to "how fast did this work?" — the platform needs to preserve both the zone-level and the aggregate reading, not collapse them into a single line.
- **Crosswalk registry**: a small table mapping each source's native ID/grid system to `entity_id`. This *is* the identity resolution layer — keep it as its own service/module, not buried in ETL scripts, because every new source plugs into it the same way.
- **Time grain**: daily vs. monthly. ONI is monthly; SST anomaly can be daily. Pick the coarsest common grain for v0 (monthly) to avoid premature complexity.
- **Stressor type**: don't let the timeline flatten every variable into "a number over time." A prototype using Great Barrier Reef and Mesoamerican Reef data surfaced a real distinction: thermal stressors (SST anomaly, ONI) build over weeks to months and propagate at basin scale, while mechanical stressors (cyclones, storms) hit in days and are local to a track. They damage reefs differently, they have different recovery timelines, and — critically — they can *confound each other's data*: a major storm can disrupt the survey window needed to measure bleaching cleanly (this is exactly what happened at the Mesoamerican Reef in 1998 with Hurricane Mitch, and at the Great Barrier Reef in 2017 with Cyclone Debbie landing on top of a bleaching event). The entity model needs `stressor_type` as a first-class field so the platform can reason about *why* two disturbance channels might be confounded in a given year, not just that a data gap exists.
- **Stressor vs. response**: bleaching alert level and DHW are still measurements of the *heat*, not of the reef's condition. The prototype added a genuinely different variable class — ecological outcome data (coral cover from AIMS's Long-Term Monitoring Program, fish biomass and coral cover from the Healthy Reefs Initiative) — and it surfaced something a stressor-only model would miss entirely: at the Mesoamerican Reef, live coral cover kept declining from 2022 to 2024 (climate-driven) while fish biomass rose sharply over the same window (management-driven, from stronger fishing enforcement). Two response variables, same entity, same period, opposite direction — because they respond to different drivers. Collapsing them into one "reef health" score would hide exactly the distinction a manager needs. `variable_class` (stressor vs. response) needs to be a first-class split, not an afterthought, so the platform can correlate a *specific* stressor against a *specific* response instead of vaguely relating "conditions" to "health."
- **Response lag differs from stressor lag**: the ONI→bleaching lag (2–4 months, Section 1) is not the same lag as bleaching→coral-mortality. Coral death from a bleaching event isn't always visible until the *following* year's survey — recovery or mortality plays out over 12+ months, not weeks. A platform that only models the fast stressor-to-stressor lag (heat→bleaching) will systematically miss the slower response-to-response lag (bleaching→cover loss) that matters more for long-term management decisions.

---

## 4. Pipeline architecture

```
[Source APIs/files] → [Ingest] → [Normalize to entity grain] → [Entity Timeline Store]
                                                                          │
                                                          [Lag-Alignment Engine]
                                                                          │
                                                     [Pattern Detection / Correlation]
                                                                          │
                                                       [Viz / Alerting layer]
```

**Ingest**
- Pull NOAA Coral Reef Watch + ONI + bleaching records (all public, no auth friction — this is a real advantage over customer data, which is siloed by *ownership*; climate data is siloed mostly by *format fragmentation*)
- Store raw pulls as-is (don't transform on ingest — keep raw/normalized separated so you can re-derive)

**Normalize**
- Resample/crosswalk each source to `entity_id` + monthly timestamp
- Land in a single long-format table: `(entity_id, timestamp, source, variable, value)` — this is your unified timeline, the direct equivalent of Amperity's unified customer event stream

**Lag-Alignment Engine** (the novel part)
- For a candidate cause variable (SST anomaly) and effect variable (bleaching alert level), test a range of lags (e.g. 0–6 months)
- At each lag, compute correlation / simple regression between cause(t) and effect(t+lag)
- Output: best-fit lag + strength, per entity and pooled across entities
- This generalizes directly to your other example (air pollution → crop yield) — same engine, different variable pair

**Pattern detection**
- v0: correlation/regression at best lag, per entity and cohort
- v1+: this is where you'd bring in more rigorous causal inference (climate science already has teleconnection indices and statistical tooling — lean on that rather than reinventing it) instead of correlation-only claims

**Viz**
- One reef site: timeline chart with SST anomaly, ONI, and bleaching alerts on shared time axis, lag visually indicated
- Cohort view: which reef sites show the strongest lagged relationship — this is your "segment" equivalent

---

## 5. Suggested stack for prototype speed

- **Ingest/ETL**: Python (pandas/xarray — xarray specifically because climate data is often gridded/netCDF, not tabular)
- **Storage**: start with Parquet files or a single Postgres/DuckDB instance — no need for a warehouse at MVP scale (a handful of reef sites × monthly data is tiny)
- **Lag-alignment + correlation**: plain Python (numpy/scipy), no need for anything fancier yet
- **Viz**: a lightweight interactive dashboard — this is a good candidate for a Claude-built artifact once real data is pulled

---

## 6. Phased roadmap

| Phase | Scope | Proves |
|---|---|---|
| **0 (this doc)** | Architecture | Shared understanding of entity model + lag engine |
| **1** | 5–10 reef sites, 3 sources, monthly grain | The full loop works end to end |
| **2** | Add raw satellite (MODIS SST) instead of pre-aggregated NOAA product | Handling gridded/netCDF ingestion, real resampling |
| **3** | Second entity type (e.g. farm parcels + air quality + crop yield) | Entity model generalizes across domains |
| **4** | IoT sensor ingestion, daily grain | Real-time / higher-frequency joins |
| **5** | Global scale, many entity types | The actual "Amperity for climate" platform |

**Recommendation: stop after Phase 1 or 2 for a pitch/demo.** That's enough to show the entity model, the lag-alignment engine, and one compelling visualized pattern — the core of the idea — without the cost of building real infrastructure.

**Phase 2 validation from the prototype:** already de-risked ahead of schedule. NOAA Coral Reef Watch's daily per-site SST/DHW record (1985–2026, three named virtual stations: Central GBR, Belize, Quintana Roo) was ingested directly and resampled from ~15,000 daily rows per site down to one peak-thermal-stress value per bleaching-season window — exactly the daily-to-event-grain resampling step Phase 2 was scoped to prove out. Two things worth carrying forward: (1) the raw files are large enough that pulling them through a live API/fetch layer hit practical limits — for a real Phase 2 build, plan on a batch/download ingestion path rather than a live per-query fetch for daily satellite products; (2) resampling to "peak DHW in a defined season window" is itself a design decision worth making explicit in the pipeline (Section 4) — the window boundaries materially affect which value gets surfaced as "the" entity-level reading for that year.

---

## 7. Citations are not data — a case where testing the claim changed the finding

The prototype initially relied on published papers' conclusions about *why* the GBR bleached in 2017 and 2022 without El Niño (reduced local wind, reduced cloud cover), and about the Atlantic Meridional Mode (AMM) explaining the Caribbean's 2017 event. Both were tested by actually ingesting the underlying station/index data instead of trusting the citation:

- **Wind (GBR):** Davies Reef weather station (AIMS, 30-min record since 1991) was unified into the GBR entity timeline and checked in the 21 days before each bleaching year's verified peak DHW date, against the 35-year climatology. Result: **3 of 7 bleaching years (2016, 2017, 2020) showed genuine anomalous calm; 1998, 2002, and 2024 didn't; 0 of 2 non-bleaching control years (2006, 2011) were calm.** More nuanced than the citation implied — calm wind is a real, recurring contributing factor, not a reliable standalone predictor, and it doesn't explain most bleaching years on its own.
- **AMM (MAR):** the Vimont/UW-Madison AMM index (monthly since 1948) was checked against all six MAR bleaching years' peak months. Result: **every single year showed AMM-SST well above the seasonal climatology, and 2017 — the year the citation specifically named — was actually the weakest signal of the six (+2.94 vs. +6.11 in 2010).** This is a *better* finding than the citation gave: AMM isn't a special explanation reached for because 2017 lacked an El Niño, it's a consistently elevated driver across MAR bleaching generally.

Two tests, two different outcomes — one partially confirmed a citation and added nuance, one confirmed a citation but revealed it was too narrowly stated. Both are more valuable than either citation alone: this is exactly the platform's core value proposition working as intended. An unverified literature claim is a hypothesis, not a finding, until it's checked against ingested data, and checking it can go either way. A production system should visibly distinguish "cited claim" from "platform-verified against ingested data" in its UI, since they carry very different confidence — see Section 8.6 for how this is represented in the schema (`provenance.verification_status`).

**A third, unplanned confirmation showed up while raw-ingesting the coral trout stock assessment.** That source is a government technical report, not a dataset built for this platform — but its own authors independently wrote the platform's thesis into their limitations section: their population model doesn't include bleaching, cyclones, or SST as inputs, and they call that "a major limitation of this model if these variables are changing systematically over time." They separately confirm, in their own analysis, the exact point-vs-underlying-signal distinction this document already draws in Section 3 — commercial catch-rate dips after a cyclone don't reflect real stock decline, since survey data (a different signal entirely) never shows the same dip, and catch rates "come back strongly one or two years after the cyclone." Neither observation was solicited; both came from reading the primary source closely enough to extract its data tables. Worth remembering when scoping future source ingestion: the case for unification sometimes turns up already made, in the source's own words, once you go past the summary figure to the underlying document.

---

## 8. Sources, schemas, and transformations — as actually built

Everything in this section reflects data genuinely ingested and processed during the prototype, not a hypothetical design. Real raw formats, real transformations, real numbers.

### 8.1 Source catalog

| Source | Entity coverage | Variable(s) | Format | Cadence | Ingestion status |
|---|---|---|---|---|---|
| NOAA CPC — Oceanic Niño Index | Global (single index, not per-entity) | `oni_index` | Fixed-width ASCII, seasonal | Monthly, 1950–present | **Raw-ingested** (direct fetch) |
| NOAA Coral Reef Watch — daily virtual station | Per-site (Central GBR, Belize, Quintana Roo) | `sst_anomaly`, `dhw`, `bleaching_alert_area` | Whitespace-delimited daily text | Daily, 1985–present | **Raw-ingested** (user download + upload — live fetch hit a 1988 truncation wall) |
| AIMS — Davies Reef weather station | Point (Central GBR) | `wind_speed` | CSV, QC-flagged | 10-min, 1991–present | **Raw-ingested** (user download via AIMS Data Explorer — live portal blocks automated fetch, `robots.txt`) |
| Vimont/UW-Madison — AMM index | Basin-level (Atlantic) | `amm_sst`, `amm_wind` | Fixed-width text | Monthly, 1948–present | **Raw-ingested** (user download — live fetch blocked, `robots.txt`) |
| NOAA IBTrACS — global tropical cyclone best-track | Per-event, unified by proximity to entity coordinate | `cyclone_category` (USA_SSHS at closest approach), position, wind, pressure | CSV, 3-hour intervals | Continuous, 1980–present (subset used) | **Raw-ingested** (user download of the full `since1980` archive, 308,468 rows, 47 seasons — a first download attempt via an alternate ERDDAP mirror returned only 2022–2025 despite matching filename conventions, a real lesson in verifying actual date coverage rather than trusting a dataset's name) |
| AIMS Long-Term Monitoring Program | Per-reef manta-tow (2,647 survey records, all reefs), rollable to any radius | `coral_cover_pct` (`MEAN_LIVE_CORAL`), plus COTS counts and coral trout sighting counts in the same file | CSV, via AIMS data API | Annual survey, 1993–2023 (this file) | **Raw-ingested** — the hardest source to get in this project: the contact-gated portal produced two rounds of metadata-only exports before the actual direct-download API link (buried inside the second metadata record, not the portal's main flow) was found and worked. See the finding below — worth the effort, since it directly overturned a cited figure. |
| Healthy Reefs Initiative / AGRRA report cards | MAR-wide, per-site data exists | `coral_cover_pct`, `fish_biomass_index`, `reef_health_index` | Site-level data via an ArcGIS Hub "Data Explorer" | Biennial | **Cited only, but confirmed real and closer than assumed** — a live per-site data portal exists (AGRRA/HRI Mesoamerican Reef Data Explorer on ArcGIS Hub, with the standard ArcGIS Hub CSV-export API pattern confirmed to exist generically), but the specific feature-service item ID wasn't located within a reasonable search budget. Different failure mode than the others in this table: not blocked, not form-gated — just not yet found. |
| Queensland DAF — coral trout Stock Synthesis assessment | GBR-wide (blue/green region split) | `fish_biomass_pct_unfished`, standardised catch rate, UVS index | PDF technical report, data tables in appendices | Periodic (2014, 2019, 2020, 2022) | **Raw-ingested** — no API or CSV existed; the primary technical report PDF itself (freely downloadable, no login/form) contains the real model-output tables. Corrected two numbers already in this document (2011: 46%→51%; the 2022 figure of 60% was actually 2020's 59%, from an earlier report than assumed) and surfaced a strong, unprompted validation of this platform's premise — see Section 7 addendum below. |

**This table is itself the finding, not just documentation.** When directly challenged on why only 4 of 8 sources were unified, checking revealed that IBTrACS — assumed blocked the same way wind and AMM initially were — was actually retrievable, just from a different mirror than the one that failed first. Unifying it didn't just fill a gap, it **changed conclusions already in this document**: two storms previously cited at Category 5 (Larry, Yasi) were Category 4 at their actual closest approach to the entity coordinate — the citation gave each storm's lifetime peak, not its entity-relevant intensity, the same point-vs-aggregate distinction Section 3 already flagged for the coral trout fishery data. It also surfaced that 2005 and 2010 — both previously flagged MAR data-confidence gaps — each had *two* real storms nearby, not the one each had been cited with, and that Hurricane Eta (2020) is genuinely absent from the entity-linked results because it hit Honduras, a real country in the Mesoamerican Reef system that this prototype's two entities (Belize, Quintana Roo) don't cover.

AIMS LTMP was harder — the contact-gated portal produced two rounds of metadata-only exports (the catalog record, not the data) before the real direct-download link was found — but unifying it produced the same result: **it disagreed with its own citation.** The previously-cited "Central GBR" 2009 coral cover figure was 19.8%; raw-ingested manta-tow data for the same year and region gives 11.6% (150km-radius mean, 7 reefs) or 21.0% (Davies Reef alone) — a roughly 2x spread depending purely on grain choice, not an error on either side. AIMS's official regional statistic uses a different reef-selection/weighting methodology than a simple radius-based average, and neither is "more correct" than the other without specifying which question is being asked.

**The remaining "cited only" sources (HRI, coral trout stock) haven't been checked yet — the lesson repeated twice now is that "cited" should be read as "not yet verified," not "verified to be unavailable," and that verification tends to change the finding, not just add confidence to it.** A production platform needs an explicit backlog of "known sources not yet unified" that gets worked down over time — not a one-time architecture decision that quietly calcifies into "this is just how it is." See Section 9.

The `Ingestion status` column matters as much as anything else in this table — it's the difference between a platform-verified finding and a repeated citation, and Section 6's wind/AMM test is the concrete proof that this distinction changes conclusions.

### 8.2 Input schemas (raw, as received)

Five representative examples — every source has its own quirks that the ingestion layer needs to absorb.

**ONI (fixed-width, seasonal rows):**
```
 SEAS  YR   TOTAL   ANOM
  NDJ 2023  28.55   1.99
  DJF 2024  28.38   1.84
```
Quirk: rows are labeled by 3-month season code (`NDJ`, `DJF`, ...), not a calendar month — needs a season→month lookup (Section 8.6) before it can align with anything else.

**Coral Reef Watch (whitespace-delimited daily, 10 columns):**
```
YYYY MM DD SST_MIN SST_MAX SST@90th_HS SSTA@90th_HS 90th_HS>0 DHW_from_90th_HS>1 BAA_7day_max
2017 03 27 29.8300 30.7200 29.0500      1.6700       1.0000    10.5000            3
```
Quirk: header repeats mid-file after metadata blocks; naive parsing must filter to rows with exactly 10 numeric fields, or silently drop all data (this happened once during the prototype — first parse returned 0 rows from a column-count mismatch).

**NOAA IBTrACS (CSV, wide — 160+ columns in the full schema, 10 used here):**
```
SID,SEASON,NUMBER,BASIN,SUBBASIN,NAME,ISO_TIME,NATURE,LAT,LON,...,USA_WIND,...,USA_SSHS,...
2017068S12148,2017,10,SP,MM,DEBBIE,2017-03-27 18:00:00,TS,-19.8,148.7,...,90,...,4,...
```
Quirks: (1) a units row immediately follows the header (`,Year,,,,,,,degrees_north,degrees_east,...`) and must be skipped, not parsed as data; (2) the file has no built-in entity linkage at all — every row is a raw lat/lon track point, and "which entity does this storm affect" has to be computed via proximity (Section 8.4), not looked up; (3) `USA_SSHS` at any given row is the category *at that specific point*, not the storm's lifetime peak — a storm can carry Category 5 in one row and Category 2 two hundred kilometers later, so "the storm's category" is meaningless without specifying *where*. This directly produced the Larry/Yasi correction in Section 8.1: the citation search reported the storm's peak-anywhere category, IBTrACS gave the category specifically where it passed the entity.

**AIMS Davies Reef (CSV, QC-flagged):**
```
time,parameter,qc_flag,qc_value
2017-03-15T00:00:00Z,Wind Speed,Good,22.4
```
Quirk: every reading carries its own QC flag — rows not flagged `Good` must be excluded, not averaged in, or contamination from sensor faults silently biases the daily mean.

**AMM (fixed-width monthly, two components per row):**
```
YEAR  MONTH   SST    WIND
2017    10    2.94    0.62
```
Quirk: two physically different variables (SST-based and wind-based mode components) share one file — they need to land as two separate `variable` rows in the unified timeline, not one.

**AIMS LTMP manta-tow (CSV, per-reef per-survey):**
```
REEF_NAME,LATITUDE,LONGITUDE,REPORT_YEAR,MEAN_LIVE_CORAL,TOTAL_TROUT,MEAN_TROUT_PER_TOW,...
DAVIES REEF,-18.82,147.64,2009,21.0,...
```
Quirks: (1) this is the only source in the catalog with no fixed entity grain at all — it's one row per reef per year, meaning "coral cover for the GBR entity" doesn't exist in the file, only "coral cover for 49 individual reefs," and the entity-level number is a design decision the ingestion layer makes, not something the source provides; (2) survey coverage is uneven year to year (7 reefs surveyed near the entity in 2009, 37 in 2002) — an unweighted mean across a different number and mix of reefs each year is a real source of apparent volatility that isn't necessarily ecological; (3) the file bundles two unrelated variable types (coral cover, coral trout sightings) the same way AMM bundles SST and wind — same transformation rule applies.

### 8.3 Working / unified schema — populated example

Extending the `LocationEntity` model from Section 3 with real 2017 GBR data, showing what a populated timeline entry actually looks like after ingestion:

```json
{
  "entity_id": "gbr_central",
  "entity_type": "reef_tract",
  "geometry": { "type": "point", "lat": -18.3, "lon": 147.7 },
  "ocean_basin": "Pacific",
  "crosswalk_keys": {
    "noaa_crw_station": "gbr_central",
    "aims_weather_station": "davies_reef"
  },
  "timeline": [
    {
      "timestamp": "2017-03-27",
      "source": "noaa_coral_reef_watch",
      "variable": "dhw",
      "variable_class": "stressor",
      "stressor_type": "thermal",
      "value": 10.5,
      "confidence": { "verification_status": "raw_ingested", "grain": "daily_satellite" }
    },
    {
      "timestamp": "2017-03-01_to_2017-03-27",
      "source": "aims_davies_reef",
      "variable": "wind_speed_anomaly",
      "variable_class": "stressor",
      "stressor_type": "thermal_precondition",
      "value": -4.16,
      "unit": "km/h_vs_climatology",
      "confidence": { "verification_status": "raw_ingested", "grain": "21day_window_mean" }
    },
    {
      "timestamp": "2017",
      "source": "aims_ltmp",
      "variable": "coral_cover_pct",
      "variable_class": "response",
      "value": null,
      "confidence": { "verification_status": "cited", "note": "no exact 2017-specific figure found; trend only" }
    }
  ]
}
```

Three things this example makes concrete: (1) `timestamp` isn't always a single day — the wind reading is genuinely a windowed aggregate, and the schema needs to carry that instead of forcing a fake single date; (2) `confidence.verification_status` is the field that operationalizes Section 6's citation-vs-ingested distinction — a UI can filter or visually flag on this directly; (3) a `null` value with a `cited` status and a note is itself useful information (a known gap, not a missing field).

### 8.4 Transformations actually applied

- **Season-code → month lookup (ONI):** `DJF→Jan, JFM→Feb, ..., NDJ→Dec`, with the season's label year kept as the row's year (matches NOAA's own convention; verified against the source rather than assumed).
- **Daily → event-window resampling (Coral Reef Watch, wind):** for each entity-year, the "bleaching season window" is defined as **December 1 (prior year) through May 31 (event year)** for GBR (Southern Hemisphere summer) and a separately-defined window would be needed for MAR (Northern Hemisphere late-summer/fall, as the 2017 finding showed — Belize/Quintana Roo peaked in October, not March). This is exactly the "window boundaries are a design decision" point from Section 6 — a single hardcoded window would have silently mis-scored the MAR entities.
- **21-day pre-peak wind aggregation:** once the actual verified DHW peak date is known (itself an output of the daily→event resampling above), wind is aggregated over the 21 days immediately preceding it — a second, dependent resampling step, chained off the first.
- **Unit normalization:** wind arrived in km/h (AIMS) — no conversion needed here, but the schema's `unit` field exists specifically because a second wind source (e.g., a future ERA5 ingestion, typically m/s) would otherwise silently corrupt comparisons.
- **QC filtering:** any row not flagged `Good` (wind) is dropped before aggregation, not averaged in — confirmed necessary in practice, not theoretical.
- **Proximity-based entity linkage (IBTrACS):** unlike every other source in this prototype, storm tracks arrive with no entity ID at all — just a raw lat/lon per 3-hour observation. Linking a storm to an entity means computing distance from every track point to every entity coordinate and keeping points within a threshold (300km was used here — wide enough to capture regional wind-field effects on a reef, though this is a real, somewhat arbitrary design choice that should be validated against actual reef damage radii rather than assumed). For each entity-storm pair that clears the threshold, the category (`USA_SSHS`) is taken from the point of *closest approach*, not the storm's global maximum — the source that most concretely forced the "grain changes the answer" lesson from Section 3, since it directly overturned two category numbers already cited elsewhere in this document.

### 8.5 Entity resolution / crosswalk table

The concrete version of Section 3's "crosswalk registry," as it actually looks with the entities used in this prototype:

| entity_id | name | lat | lon | ocean_basin | noaa_crw_key | aims_weather_key | hri_region_key |
|---|---|---|---|---|---|---|---|
| `gbr_central` | Central Great Barrier Reef | -18.3 | 147.7 | Pacific | `gbr_central` | `davies_reef` | — |
| `mar_belize` | Belize Barrier Reef | 17.0 | -87.8 | Atlantic | `belize` | — | `belize` |
| `mar_qroo` | Quintana Roo, Mexico | 20.0 | -87.0 | Atlantic | `quintana_roo` | — | `mexico` |

Two real gaps this table exposes honestly: no AIMS-style weather station key exists for either MAR entity (which is exactly why the wind hypothesis could only be tested at the GBR — MAR's "calm wind" claim from the original citation search remains unverified), and the HRI region key doesn't cleanly map 1:1 to the satellite-derived site names (HRI reports at a coarser "country/region" grain than NOAA's site-level virtual stations) — a real instance of the grain-mismatch problem from Section 3.

### 8.6 Lookup tables

**ENSO state classification** (from ONI value → label, standard NOAA convention, used throughout the demo):
| ONI range | Label |
|---|---|
| ≥ +1.5 | Strong El Niño |
| +0.5 to +1.49 | El Niño |
| -0.49 to +0.49 | Neutral |
| -1.49 to -0.5 | La Niña |
| ≤ -1.5 | Strong La Niña |

**Ocean basin → relevant teleconnection index** (the direct fix for Section 1's "one global driver doesn't generalize" finding):
| Ocean basin | Primary index | Secondary index |
|---|---|---|
| Pacific | ENSO / ONI | PDO |
| Atlantic | AMM | AMV/AMO |
| Indian | IOD | — |

**Cyclone/hurricane category** (Saffir-Simpson, used identically for both Pacific cyclones and Atlantic hurricanes in the entity timeline — same `stressor_type: mechanical` variable, different `source`):
| Category | Sustained wind |
|---|---|
| 1 | 119–153 km/h |
| 2 | 154–177 km/h |
| 3 | 178–208 km/h |
| 4 | 209–251 km/h |
| 5 | ≥ 252 km/h |

**`variable_class` taxonomy** (Section 3's stressor/response split, as actually used):
| variable_class | stressor_type | Examples used |
|---|---|---|
| stressor | thermal | `sst_anomaly`, `dhw`, `oni_index`, `amm_sst` |
| stressor | thermal_precondition | `wind_speed_anomaly` (contributes to thermal stress without being heat itself) |
| stressor | mechanical | `cyclone_category` |
| response | — | `coral_cover_pct`, `fish_biomass_index`, `bleaching_severity_pct` |

---

## 9. Open questions worth deciding before Phase 1 build

- Confidence/provenance: how do you represent that a value came from a 5km satellite pixel resampled to a reef polygon vs. an in-situ buoy reading? This matters for trust in any "pattern" you surface. **Update from the prototype:** this isn't hypothetical — the Mesoamerican Reef entity has real, quantified severity data for some years (2015–16, 2023) and only qualitative or missing data for others (1998, 2005, 2010, 2017), and two of those gap years are exactly the years a major hurricane hit. The platform should be able to surface *why* a confidence gap exists (storm disruption, younger monitoring network) by checking other stressor sources on the same entity/timeframe — not just flag "no data."
- Missing data handling: satellite/IoT data has gaps (cloud cover, sensor outages, cost) — decide interpolation policy early, it will bias lag correlations if done carelessly.
- Where causal claims stop and correlation claims start — worth stating explicitly in any output/UI so the platform doesn't overclaim (this is a genuine risk with "look, a pattern!" tools).
- Multi-stressor disambiguation: when thermal and mechanical stressors hit the same entity in the same window (Hurricane Mitch + 1998 MAR bleaching; Cyclone Debbie + 2017 GBR bleaching), the platform needs a way to represent "these two things happened together and are hard to separate" rather than silently attributing all the damage to whichever source got queried first.
- **Ingestion completeness as an ongoing discipline, not a one-time decision.** The prototype's own source catalog (Section 8.1) initially mislabeled 4 sources as "cited, not raw-ingested" in a way that implied a natural category boundary. Directly checking two of them found real, public, structurally-ready datasets (IBTrACS for storms; AIMS LTMP transect data for coral cover) that had simply never been requested — not genuine access barriers like the ones that initially blocked wind and AMM. A production platform needs a visible, tracked backlog of "known sources not yet unified," reviewed periodically, so partial coverage doesn't quietly become permanent coverage. The uncomfortable meta-lesson: a platform whose whole premise is unifying scattered sources needs active pressure testing of its own claimed source list, not just its computed patterns — it's easy to build one unification pipeline, prove it works, and stop, while the "not yet done" pile keeps growing unexamined.
