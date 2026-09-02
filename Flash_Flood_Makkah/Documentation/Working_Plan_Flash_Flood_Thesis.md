# Working Plan — Flash Flood Susceptibility Mapping (Wadi Mehassar)

A month-by-month execution plan for the proposal. Each phase lists concrete tasks, the tool used, and the deliverable that closes it out — so you always know what "done" looks like before moving on.

---

## Phase 0 — Setup (Week 1, before Month 1 officially starts)

1. **Get basin boundary and coordinate system settled first.** Delineate the Wadi Mehassar watershed from the SRTM DEM (ArcGIS Pro Hydrology toolset or QGIS SAGA `Watershed Basins`) using a pour point at the outlet (~218 m). Reproject everything to one projected CRS (UTM Zone 37N, EPSG:32637 fits this longitude) — every raster and vector layer for the rest of the project uses this CRS, no exceptions.
2. Set up your Python environment once: `conda create -n floodml python=3.11`, then `scikit-learn`, `xgboost`, `rasterio`, `geopandas`, `rasterstats`, `shap`, `matplotlib`. Test it opens a raster before you rely on it later.
3. Create the folder structure you'll use for nine months: `/data/raw`, `/data/processed`, `/factors`, `/samples`, `/models`, `/outputs/maps`, `/outputs/figures`, `/scripts`. Unglamorous but saves real time in Month 5 when you're not hunting for files.
4. Open a lab notebook (even a shared Google Doc) where every data-processing decision gets one line: source, date accessed, any resampling or clipping applied. You will need this for your methods chapter — write it down now, not in Month 8.

**Deliverable:** basin boundary shapefile, working Python environment, folder structure, decision log started.

---

## Phase 1 — Data Collection (Months 1–2)

1. **DEM:** Download SRTM 30 m tiles covering the basin (USGS EarthExplorer or OpenTopography). Clip to basin boundary + a small buffer (~500 m) so edge effects don't bite you during slope/TWI calculation.
2. **Satellite imagery:** Pull Sentinel-1 (SAR, for later flood mapping) and Sentinel-2 (optical, for land cover / NDVI) via the Copernicus Data Space or Google Earth Engine. For Sentinel-1, identify and download scenes bracketing your target flood event(s) — one pre-event, one during/immediately post-event.
3. **Rainfall:** Contact or search NCM (National Center for Meteorology) for station data near the basin; if station access is slow, fall back to published intensity–duration–frequency values from the El-Saoud & Othman (2022) paper or similar literature as a documented placeholder — flag this in your decision log so it's not a silent assumption later.
4. **Soil/geology:** Source existing published geological maps for the Makkah region (Saudi Geological Survey publications are the standard reference).
5. **Population:** Collect (a) national census figures for the permanent urban districts in the basin, and (b) published Hajj capacity/occupancy figures for Mina, Muzdalifah, and Arafat — these will likely come from Hajj Ministry reports or peer-reviewed papers reporting pilgrim distribution.
6. **Reference flood data:** Get the published HEC-RAS inundation extent (from El-Saoud & Othman 2022 or equivalent) — you'll need this as a raster or digitized polygon for Section 6.7 validation. Also confirm your 1–2 target Sentinel-1 flood dates now, not in Month 3, since Sentinel-1 revisit gaps can force you to change target events.

**Deliverable:** every raw dataset sitting in `/data/raw`, each with a one-line provenance note in the decision log. This is the single most schedule-risky phase — a missing dataset here delays everything downstream, so don't move to Phase 2 with placeholders you haven't actually resolved.

---

## Phase 2 — Conditioning Factor Preparation (end of Month 2 – early Month 3)

1. Derive **elevation, slope, TWI** from the clipped DEM (Slope: `gdaldem slope`; TWI: SAGA `Topographic Wetness Index` or a manual flow-accumulation/slope combination).
2. Derive **drainage density** and **distance to main wadi channels**: extract the stream network from the DEM (flow accumulation threshold), then `gdal_proximity` or Euclidean Distance for the distance layer, and a moving-window line-density calculation for drainage density.
3. Classify **land use/land cover** from Sentinel-2 (supervised classification — random forest or a simpler maximum-likelihood classifier is fine for this factor since it's an input, not your main model).
4. Compute **NDVI** from Sentinel-2 red/NIR bands.
5. Resample every factor layer to the same 30 m grid, same extent, same CRS as the DEM. This step is where most beginner mistakes happen — a one-pixel misalignment between factors silently corrupts every sample point you extract later. Verify alignment with `rasterio` by checking `.transform` and `.shape` match across all layers before proceeding.
6. Run the **VIF/correlation check** now, on the physical factors, while you still have time to drop or combine a redundant one without reworking the whole sample table later.

**Deliverable:** eight aligned, resampled factor rasters in `/factors`, VIF results logged.

---

## Phase 3 — Hajj Population Density Layer + Flood Inventory (Month 3)

This is the phase that makes the thesis original, so budget real time here rather than treating it as routine.

1. Digitize or source zone boundaries for Mina, Muzdalifah, and Arafat encampment/tent-block areas from Sentinel-2 imagery or published camp-planning maps.
2. Redistribute your published occupancy figures across each zone's built/usable footprint (not the full administrative area) — this is the dasymetric step described in Section 6.3 of the proposal. Document the exact capacity source and footprint mask used for each zone.
3. Rasterize the result to your standard 30 m grid.
4. In parallel: classify your 1–2 chosen Sentinel-1 scenes into binary flood/non-flood (simple SAR backscatter thresholding — flooded surfaces typically show low backscatter; visually cross-check against the HEC-RAS reference extent from Phase 1).
5. Generate balanced flood/non-flood sample points from the classified raster, extract all nine factor values (eight physical + Hajj density) at each point into a single table.
6. Set aside a **spatially contiguous test region** now (e.g., one sub-catchment held out entirely), not a random 30% — this is what Section 6.4 commits to, and it's much easier to reserve a clean spatial block before you've built momentum on model training.

**Deliverable:** Hajj density raster, flood/non-flood sample table with all factor values attached, spatial train/test split defined and saved as two separate point files.

---

## Phase 4 — Model Training (Months 4–5)

1. Train Random Forest and XGBoost on the **full factor set** (all 9 factors including Hajj density) using the spatial train split.
2. Train a second RF/XGBoost pair on the **physical-factors-only set** (drop Hajj density) — your baseline for the with/without comparison.
3. Tune each model modestly (grid search on `n_estimators`, `max_depth` — don't over-invest here; a defensible, well-validated basic model beats an over-tuned fragile one for a thesis timeline).
4. Predict probability surfaces across the whole basin for all four models (RF-full, XGB-full, RF-baseline, XGB-baseline).
5. Reclassify each probability surface into 5 classes (Natural Breaks, applied consistently across all four surfaces using the same break points where possible for comparability).
6. Extract variable importance from both full models; compare rankings side by side.

**Deliverable:** four trained models, four susceptibility maps (raw probability + 5-class), variable importance table.

---

## Phase 5 — Comparison, Validation, Analysis (Months 6–7)

1. **With/without comparison (Section 6.6):** difference the full vs. baseline susceptibility class rasters; cross-tabulate how many pixels shifted class, and where spatially (map it — this is likely your headline figure).
2. **Accuracy metrics:** ROC–AUC, accuracy, precision, recall, F1 on the held-out spatial test set, for both full and baseline models.
3. **HEC-RAS comparison:** overlay your high/very-high susceptibility zones against the published HEC-RAS inundation extent; report spatial agreement (simple overlap percentage or Cohen's kappa if you want a formal statistic).
4. Draft your key results figures now, while the analysis is fresh: susceptibility map, variable importance chart, with/without comparison map, HEC-RAS overlay.

**Deliverable:** all validation numbers, all core figures, a one-page results summary you can hand to your supervisor for a checkpoint before writing begins in earnest.

---

## Phase 6 — Writing and Thesis Preparation (Months 8–9)

1. Write Methods first (you already have the decision log and the proposal's Section 6 as scaffolding — this chapter should be the fastest to draft).
2. Write Results directly from the Phase 5 deliverable — figures and numbers are already in hand, so this is assembly, not new analysis.
3. Write Discussion around your three research questions specifically — answer each one explicitly with the evidence you generated (with/without comparison → RQ1, importance rankings → RQ2, HEC-RAS overlap → RQ3).
4. Introduction, Abstract, and Conclusion last, once you know exactly what you found.
5. Leave 2–3 weeks of the 9-month window unallocated as buffer — data-access delays in Phase 1 or a stubborn alignment bug in Phase 2 are the two most common places theses like this slip, and buffer time is cheaper to plan for now than to find later.

**Deliverable:** complete draft thesis, ready for supervisor review.

---

## Two schedule risks worth flagging now

- **Rainfall and Hajj occupancy data access (Phase 1)** — these are the two datasets most likely to require official requests with lead time. Start these requests in Week 1, not Month 1.
- **Sentinel-1 flood event selection (Phase 1/3)** — confirm scene availability for your target flood date(s) before committing to them in the proposal defense; SAR revisit gaps can mean the "ideal" flood event has no usable pre/post scene pair.
