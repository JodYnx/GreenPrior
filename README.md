# GreenPrior (جرين برايور)

**An explainable, satellite-based decision-support system that helps authorities decide where to start greening assessments first in Ha'il, Saudi Arabia.**

[Live demo](https://green-prior.base44.app) · Built solo in 3 days for the University of Ha'il Space Innovation Challenge (World Space Week 2026), *AI & Space Data Science* track.

> GreenPrior does **not** decide where trees must be planted. It ranks areas by priority for further field assessment and explains *why* each area received its ranking.

---

## Why this project

Greening in arid regions is expensive and water-sensitive. Choosing where to begin field studies often relies on scattered data and expert judgment, which makes locations hard to compare and decisions hard to justify.

GreenPrior asks a narrower, more answerable question:

> *Which areas deserve higher priority for further greening assessment, when balancing environmental need, climatic water demand, and land eligibility?*

---

## What it does

- Splits the Ha'il study area (~30 km × 30 km) into **11,088 grid cells of ~300 m × 300 m**.
- Computes satellite-derived indicators for every cell using **summer 2026** data.
- Applies two gates (land eligibility, vegetation deficit), then ranks the remaining candidates.
- Classifies every cell as **High / Medium / Low priority**, **Excluded**, or **Insufficient Data**.
- Generates a plain-language reason for each result.
- Serves an interactive satellite-basemap map with an Arabic explanation assistant that can describe a cell or compare two cells using only the model's own table (no invented numbers, no external LLM).

### Results for Ha'il (summer 2026)

| Class | Cells | Share |
|---|---:|---:|
| High priority | 1,859 | 16.8% |
| Medium priority | 1,859 | 16.8% |
| Low priority | 6,018 | 54.3% |
| Excluded (land not eligible) | 1,173 | 10.6% |
| Insufficient data | 179 | 1.6% |
| **Total** | **11,088** | |

---

## Data sources

All sources are free and open, accessed through [Google Earth Engine](https://earthengine.google.com) (a research platform used with official NASA/ESA/FAO products, not the consumer Google Earth viewer).

| Indicator | Dataset | Earth Engine ID | Provider | Native resolution |
|---|---|---|---|---|
| Heat exposure (land surface temperature) | MODIS Terra MOD11A1 V6.1 | `MODIS/061/MOD11A1` | NASA | ~1 km |
| Vegetation (NDVI) | Sentinel-2 Surface Reflectance | `COPERNICUS/S2_SR_HARMONIZED` | ESA | 10 m |
| Land eligibility (land cover) | ESA WorldCover v200 | `ESA/WorldCover/v200` | ESA | 10 m |
| Climatic water demand (reference ET) | FAO WaPOR v3 Level 1 RET | `FAO/WAPOR/3/L1_RET_D` | FAO | ~19 km |
| Validation only | Ha'il Airport station (OEHL) hourly METAR | via Iowa Environmental Mesonet | Public archive | Point |

Time window: **1 June – 1 September 2026** (92 MODIS images, 23 Sentinel-2 images at <20% cloud, 9 WaPOR dekads).

---

## Method

### 1. Grid
A 300 m grid over the bounding box `lon 41.55–41.85`, `lat 27.35–27.65`. Longitude steps are corrected for latitude (`cos(lat)`), so cells are close to true 300 m squares. Cell IDs follow a fixed longitude-major ordering shared by all scripts.

### 2. Indicators
- **Heat score:** summer-mean land surface temperature (°C), stretched to 0–100 between the 2nd and 98th percentiles. This is *surface* temperature, not air temperature.
- **Vegetation deficit:** each cell's summer-mean NDVI is compared with the mean NDVI of its local neighborhood (circular kernel, **300 m radius ≈ 600 m span**). Deficit = neighborhood baseline − cell NDVI, stretched to 0–100 between the 5th and 95th percentiles. Naturally barren desert is therefore not flagged as a problem on its own.
- **Land eligibility:** from ESA WorldCover. A cell is eligible when built-up + water + tree-cover together cover **less than 50%** of it. Eligibility is not the same as planting suitability.
- **Water context:** mean reference evapotranspiration. Because its native resolution (~19 km) is coarser than the study area, it is used as **city-wide context**, not as a per-cell ranking factor.

### 3. Classification
1. **Gate 1 – land eligibility:** failing cells are *Excluded*.
2. **Gate 2 – vegetation gate:** eligible cells with a vegetation-deficit score below **55** are *Low*. This prevents a hot but naturally barren cell from ranking High on heat alone (a flaw found during testing).
3. **Weighted score** for the remaining candidates:
   `score = 0.45 × heat + 0.45 × vegetation deficit + 0.10 × water context`
4. Candidates at or above the candidate-pool **median** score are *High*; the rest are *Medium*.
5. Eligible cells with no valid temperature reading are labeled *Insufficient Data* rather than guessed.

### 4. Weight sensitivity
Five weight scenarios (balanced, heat-heavy, vegetation-heavy, low water, no water) were compared. Top-10 overlap between scenarios ranged from **70% to 100%**, and top-10% overlap from about **80% to 100%**. The balanced scenario was selected. The weights are a documented MVP choice, not a claim of scientific optimality. Since the water score is constant across cells, it shifts scores uniformly and does not change ranking.

### 5. Validation against a ground station
Satellite land surface temperature was compared with air temperature from the Ha'il Regional Airport station (OEHL), 2,184 hourly readings, June–August 2026:

| Measure | Value |
|---|---:|
| Satellite surface temperature (study-area mean) | 44.49 °C |
| Station daytime air temperature (9:00–15:00 mean) | 39.21 °C |
| Difference | 5.28 °C |

A hotter ground surface than air under desert sun is physically expected, so the gap supports the realism of the satellite readings. The station is a single point, so it is used for validation only, not for spatial ranking.

---

## Repository structure

```
.
├── scripts/
│   ├── 01_hail_heat_exposure.py          # MODIS LST on the 300 m grid
│   ├── 02_hail_vegetation_deficit.py     # Sentinel-2 NDVI + 600 m local baseline
│   ├── 03_hail_land_eligibility.py       # ESA WorldCover shares + eligibility
│   ├── 04_hail_water_context.py          # WaPOR RET regional context
│   ├── 05_merge_and_weight_sensitivity.py# merge, gates, 5-scenario test
│   ├── 06_greenprior_final_model.py      # frozen scoring + explanations
│   └── 07_validation_ground_station.py   # LST vs OEHL station
├── data/
│   ├── greenprior_final_model_300m.csv   # final per-cell results
│   └── OEHL.csv                          # station readings used for validation
├── docs/
│   └── GreenPrior_Presentation_AR.pdf
└── README.md
```

---

## Running the pipeline

Scripts 01–04 run in [Google Colab](https://colab.research.google.com) (internet access and Earth Engine authentication required).

1. Get Earth Engine access and a Google Cloud project with the Earth Engine API enabled.
2. In each script, set `GCP_PROJECT_ID` to your own project ID.
3. Install dependencies:
   ```bash
   pip install earthengine-api pandas numpy
   ```
4. Run scripts `01` → `04` (each writes a CSV), then `05` (optional sensitivity check) and `06` (final model).

> The grid-building block is intentionally identical in scripts 01–04. Do not change it in one script only, or cell IDs will stop lining up when files are merged.

**Security:** never commit API keys, tokens, or credentials to this repository.

---

## Limitations

- Soil, salinity, groundwater, and tree-species suitability are **not** modeled.
- Heat at 300 m reflects the ~1 km MODIS pixel beneath it; adjacent cells can share the same value.
- Water demand (RET) is spatially uniform across the study area at its native resolution, so it cannot rank individual cells.
- Land eligibility reflects land-cover share only. It does not confirm a site is suitable for planting.
- The 600 m neighborhood, the vegetation-gate threshold (55), and the weights are practical MVP choices, not proven universal optima.
- Validation uses a single ground station.
- The results are a **prioritization for further assessment**, not a final ecological or engineering decision.

---

## Data that requires formal access

The National Center for Meteorology (NCM) operates an automated weather station API that would improve ground validation. Access requires a formal application, review, and approval, so it is documented here as a next step and was not used in this version.

---

## Roadmap

- Add factors: soil and salinity, groundwater, species suitability.
- Use official national station data (NCM) and multi-year data for stronger validation.
- Extend the method to other cities and regions in Saudi Arabia.
- Volunteering concept: a second portal where citizens can join planting events at high-priority sites, with hours tracked and linked to a national volunteering platform, while authorities keep the analysis map.

---

## Acknowledgments and data attribution

NASA (MODIS), ESA / Copernicus (Sentinel-2, WorldCover), FAO (WaPOR), Google Earth Engine, Iowa Environmental Mesonet (station archive), Esri World Imagery (basemap in the web app). Please follow each provider's terms of use when reusing their data.

## Author

[Your Name] · [LinkedIn / Portfolio link]

## License

[Choose a license, e.g. MIT, and add a LICENSE file.]
