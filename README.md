# Project SILAW 🛰️🌊

**Spatial Illumination and Light Analysis for Wildlife** — quantifying night-time light pollution over the Philippines' Marine Protected Areas (MPAs) and forecasting where ecological risk is heading.

An end-to-end geospatial + time-series pipeline: satellite radiance → per-MPA risk scoring → unsupervised segmentation → 5-year forecasting, built to help conservation planners answer *which* MPAs to prioritize and *how urgent* each one is.

`Python` · `Google Earth Engine` · `VIIRS DNB` · `GeoPandas` · `rasterio` · `scikit-learn` · `statsmodels (ARIMA)` · `Prophet`

---

## TL;DR — what the analysis found

- Night-time radiance over Philippine MPAs has **risen substantially from 2014 to 2023**, with a visible COVID-era dip (2020–21) and a sharp rebound in 2023 as restrictions lifted.
- On the study's constructed risk index, the *average* MPA sits in the **"critical"** band by 2023.
- K-means (k=6, silhouette **0.75**) cleanly separates the fleet into six ecological archetypes. One — **Urban-Impacted High-Radiance**, concentrated around **Cebu City** — carries far more light-pollution exposure than the rest.
- Forecasting to 2028 (Prophet vs ARIMA) projects the high-radiance urban cluster to keep climbing, while ecologically rich but remote clusters stay comparatively stable.

> ⚠️ These are findings from a heuristic risk index on a short annual series — see [Limitations](#limitations) before quoting numbers. The index is a prioritization aid, not a validated ecological risk model.

---

## Why this project

The Philippines sits at the center of the Coral Triangle — the global peak of marine biodiversity — and over 1,500 designated MPAs are meant to protect it. Light pollution is a well-documented but under-monitored stressor on marine life (coral gametogenesis, fish circadian rhythms, zooplankton vertical migration), yet most artificial-light-at-night research is terrestrial. SILAW asks a concrete, spatial question: **using open satellite data, which MPAs are most exposed to worsening light pollution, and can we anticipate it?**

## Data

| Source | What | Notes |
|---|---|---|
| **VIIRS Day/Night Band** (NOAA, via Google Earth Engine) | Monthly night-time radiance composites, 2014–2023 | Aggregated to yearly means, clipped to PH, 250 m/pixel, units nW/sr/cm² |
| **OCHA Philippines MPA dataset** (via HDX) | 339 MPAs with location, type, and environmental attributes | Cleaned to **199 valid geometries** after CRS checks and geometry validation |

All datasets are open. Spatial layers were reprojected to **Philippine Albers Equal Area (EPSG:3123)** to preserve area for radiance extraction.

## Method

```
VIIRS composites ──┐
                   ├─▶ zonal mean radiance per MPA per year
MPA polygons ──────┘            │
                                ▼
                     Risk score  R = k₁·(r̄/p) + k₂·e
                     (r̄ = mean radiance, p = harm threshold,
                      e = environmental-feature score)
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
    Risk mapping &      K-means segmentation   ARIMA / Prophet
    trend analysis      (elbow + silhouette)   5-yr risk forecast
```

**Risk index.** Each MPA's yearly score combines light exposure (radiance relative to a harm threshold of ~0.3 nW/sr/cm², drawn from the coral-ALAN literature) with an environmental-sensitivity term weighting corals, seagrass, mangroves, and reef area. Scores bin into five bands: Low ≤ 0.25 · Medium ≤ 0.5 · High ≤ 0.75 · Very High ≤ 1.0 · Critical > 1.0.

**Segmentation.** K-means on radiance + environmental features (dropping all-present / all-absent features to avoid degenerate dimensions), with **k chosen by the elbow method (k=6)** and validated by **silhouette (mean 0.75)**.

**Forecasting.** ARIMA and Prophet trained on 2014–2022, tested on 2023 by RMSE, then used to project five years forward at both the national and per-cluster level.

## Results

**Model comparison (2023 hold-out RMSE):**

| Level | ARIMA | Prophet |
|---|---|---|
| All MPAs (aggregate) | 0.2312 | **0.1579** |
| Per-cluster (mean) | **0.2067** | 0.2099 |

Prophet wins clearly on the aggregate series; per-cluster the two are near-tied (ARIMA edges it, dragged by a single high-variance cluster). Prophet was selected as the primary forecaster on balance.

**The six MPA archetypes:**

| Cluster | Archetype | Signature |
|---|---|---|
| 0 | Expansive Coral-Dominated | Largest reef area, low radiance |
| 1 | Moderate-Radiance Coastal | Mid radiance, low feature diversity |
| 2 | Diverse Coral–Seagrass | Balanced, high ecological interdependence |
| 3 | Coral-Enriched, Moderate Disturbance | Coral present, average human impact |
| 4 | Mangrove-Dominated Buffer | Mangrove-heavy natural buffer zone |
| 5 | **Urban-Impacted High-Radiance** | Highest radiance; clustered around Cebu City |

The headline decision output: **Cluster 5 is the intervention priority** (shielded lighting, dark-sky measures, urban-planning integration), while ecologically rich Clusters 0 and 2 warrant holistic, whole-ecosystem monitoring to keep them resilient.

## Limitations

Written plainly because they matter for how the results should be read:

- **The risk score is a constructed heuristic, not a calibrated model.** Its weights and the ~0.3 nW/sr/cm² harm threshold are reasoned estimates, not empirically fitted to ecological outcomes. Because the index is mechanically driven by radiance, its trend necessarily tracks radiance — so "risk is rising" and "radiance is rising" are close to the same statement. The index is best read as a *transparent prioritization tool*, not a prediction of biological harm.
- **The forecasts rest on a short annual series** (≈9 points, single-year hold-out). ARIMA/Prophet on this little data are essentially fitting a trend line with wide, honest uncertainty bands; the RMSE gap between models is small and shouldn't be over-interpreted. Monthly composites (already available) would give a far richer series and are the obvious next step.
- **~40% of MPA records were dropped** for invalid geometries, so coverage is partial and skewed toward well-digitized areas.
- **VIIRS resolution (~500 m native)** blurs small MPAs and can bleed coastal-city light into adjacent protected polygons.

## Where I'd take it next

Rebuild on the **monthly** VIIRS series (seasonality + far more signal for forecasting); validate the risk index against an external ecological outcome (e.g. reef-health or bleaching records) to move it from heuristic toward calibrated; and add spatial spillover modeling so urban light bleeding into neighboring MPAs is estimated rather than assumed.
