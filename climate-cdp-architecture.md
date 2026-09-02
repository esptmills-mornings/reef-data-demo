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
├── crosswalk_keys         (per-source IDs: NOAA station ID, satellite pixel index, etc.)
└── timeline[]
      ├── timestamp
      ├── source
      ├── variable          (sst_anomaly, chlorophyll, oni_index, bleaching_alert_level, cyclone_category, ...)
      ├── stressor_type      (thermal | mechanical | chemical | biological ...)
      ├── value
      └── confidence / provenance
```

Key design decisions to make explicit early:
- **Grain**: point vs. polygon. Reefs are naturally polygons; SST grids are pixels. You'll resample one to the other — decide which direction and document the loss.
- **Crosswalk registry**: a small table mapping each source's native ID/grid system to `entity_id`. This *is* the identity resolution layer — keep it as its own service/module, not buried in ETL scripts, because every new source plugs into it the same way.
- **Time grain**: daily vs. monthly. ONI is monthly; SST anomaly can be daily. Pick the coarsest common grain for v0 (monthly) to avoid premature complexity.
- **Stressor type**: don't let the timeline flatten every variable into "a number over time." A prototype using Great Barrier Reef and Mesoamerican Reef data surfaced a real distinction: thermal stressors (SST anomaly, ONI) build over weeks to months and propagate at basin scale, while mechanical stressors (cyclones, storms) hit in days and are local to a track. They damage reefs differently, they have different recovery timelines, and — critically — they can *confound each other's data*: a major storm can disrupt the survey window needed to measure bleaching cleanly (this is exactly what happened at the Mesoamerican Reef in 1998 with Hurricane Mitch, and at the Great Barrier Reef in 2017 with Cyclone Debbie landing on top of a bleaching event). The entity model needs `stressor_type` as a first-class field so the platform can reason about *why* two disturbance channels might be confounded in a given year, not just that a data gap exists.

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

---

## 7. Open questions worth deciding before Phase 1 build

- Confidence/provenance: how do you represent that a value came from a 5km satellite pixel resampled to a reef polygon vs. an in-situ buoy reading? This matters for trust in any "pattern" you surface. **Update from the prototype:** this isn't hypothetical — the Mesoamerican Reef entity has real, quantified severity data for some years (2015–16, 2023) and only qualitative or missing data for others (1998, 2005, 2010, 2017), and two of those gap years are exactly the years a major hurricane hit. The platform should be able to surface *why* a confidence gap exists (storm disruption, younger monitoring network) by checking other stressor sources on the same entity/timeframe — not just flag "no data."
- Missing data handling: satellite/IoT data has gaps (cloud cover, sensor outages, cost) — decide interpolation policy early, it will bias lag correlations if done carelessly.
- Where causal claims stop and correlation claims start — worth stating explicitly in any output/UI so the platform doesn't overclaim (this is a genuine risk with "look, a pattern!" tools).
- Multi-stressor disambiguation: when thermal and mechanical stressors hit the same entity in the same window (Hurricane Mitch + 1998 MAR bleaching; Cyclone Debbie + 2017 GBR bleaching), the platform needs a way to represent "these two things happened together and are hard to separate" rather than silently attributing all the damage to whichever source got queried first.
