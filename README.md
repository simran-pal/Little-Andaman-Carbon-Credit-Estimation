# Forest Carbon Stock Change Estimation - Little Andaman Island, India

### GEDI LiDAR | Sentinel-2 | Gradient Boosting | 2020-2022

Little Andaman Island sits at the center of a development debate in India. NITI Aayog's *Sustainable Development of Little Andaman Island - Vision Document* proposes opening up roughly 240 sq km, about 35% of the island, for a greenfield city pitched as a free trade hub, including an airport, film city, and tourism SEZ. The proposal involves de-reserving a third of the island's protected forest and de-notifying part of the Onge Tribal Reserve, home to a particularly vulnerable indigenous group.

This raised a question: what is actually at stake, ecologically, if development of this scale proceeds, and how that can be expressed in terms development economics already uses, tonnes of CO2e and dollars, rather than only hectares of forest.

This project builds the measurement layer underneath that question. I fused NASA GEDI LiDAR shots with ESA Sentinel-2 imagery to estimate above-ground biomass density (AGBD) across Little Andaman Island for 2020 and 2022, producing a wall-to-wall AGBD map, a change detection raster, and a carbon stock change estimate structured around Verra's VM0042 remote sensing framework. This is an exploratory feasibility study, scoped to translate ecological change into economic terms rather than to produce a submission-ready MRV report, though every methodological choice maps to a real framework decision.

A longer 2020 to 2025 longitudinal extension of this analysis is currently in progress and will be published as an update to this repository. See **Future Updates** below.

---

## Figures

| AGBD Map 2020 | AGBD Map 2022 |
| --- | --- |
| [AGBD 2020](/simran-pal/Little-Andaman-Carbon-Credit-Estimation/blob/master/figures/agbd_map_2020.png) | [AGBD 2022](/simran-pal/Little-Andaman-Carbon-Credit-Estimation/blob/master/figures/agbd_map_2022.png) |

| GEDI Tracks + Bounding Box |
| --- |
| [GEDI Tracks](/simran-pal/Little-Andaman-Carbon-Credit-Estimation/blob/master/figures/gedi_tracks.png) |

The diagonal stripes visible in both maps are GEDI orbital tracks, the model predicts wall-to-wall by interpolating S2 spectral features between sparse LiDAR ground truth. The stripes are more pronounced in 2022 due to lower shot density (4,297 vs 7,722 shots), reflecting cloud cover reducing L2A coverage. These maps show spatial patterns in predicted biomass, scoped as model output rather than spatially continuous LiDAR measurement.

---

## Key Results

| Metric | Value |
| --- | --- |
| GEDI shots (2020, after QC) | 7,722 |
| GEDI shots (2022, after QC) | 4,297 |
| Model CV R² (Gradient Boosting) | **0.791** |
| Model CV RMSE | 38.9 Mg/ha (~21%) |
| Comparable forest area | 67,374.6 ha |
| Carbon stock 2020 | 20,820,842 tCO2e |
| Carbon stock 2022 | 19,946,522 tCO2e |
| Mean AGBD change | -7.53 Mg/ha |
| Net change (2020 to 2022) | -874,143 tCO2e (decline) |
| Indicative value loss @ $5.60/tCO2e | ~$4.9M |

---

## Estimating Economic Value of Carbon Stock Change

AGBD is the starting point. The conversion chain that translates biomass on a satellite map into a monetary figure looks like this:

1. **AGBD (Mg/ha)** is the model's output: biomass per hectare of forest, in megagrams (equivalent to metric tonnes).
2. **Carbon stock (tC/ha)** is AGBD multiplied by the carbon fraction of dry biomass, taken as 0.47 per IPCC 2006 guidelines.
3. **CO2 equivalent (tCO2e/ha)** converts carbon into the standard greenhouse gas accounting unit, multiplying by the molecular weight ratio of CO2 to carbon, 44/12.
4. **Total stock** scales this up across the comparable forested area (67,374.6 ha) to an island-wide tCO2e figure.
5. **Change in stock** between the two periods is what carries economic meaning: sequestration generates creditable tonnes, decline generates a liability. The underlying model carries a cross-validated RMSE of approximately 21% of mean AGBD, meaning pixel-level predictions carry real uncertainty, while area-aggregated estimates such as the island-wide mean are comparatively more stable, since random prediction error partly cancels out at that scale. The -874,143 tCO2e net change is read with this noise floor in mind, as a meaningful shift rather than a precisely bounded one.
6. **Market value** applies a price per tonne. Voluntary carbon markets price tonnes differently depending on credit quality and buyer type, from REDD+ spot prices around $2.70/tCO2e to high-integrity credits near $14.80/tCO2e. At the VCM average of $5.60/tCO2e, the estimated decline corresponds to roughly $4.9M in indicative value.

This chain is what lets a remote sensing result, a shift in canopy height and reflectance picked up by satellites, sit in the same units used in REDD+ project design documents, carbon credit issuance, and the cost-benefit conversations around development projects like the one proposed for Little Andaman.

Verra's frameworks (VM0042 for REDD+ crediting, VM0055 for remote sensing MRV, VT0005 for the biomass estimation tool) define this conversion process structurally. This project follows that structure to express ecological change in economic terms, positioned as an exploratory feasibility study.

---

## Methodology

### Data Sources

* **NASA GEDI L4A v2.1** - aboveground biomass density (AGBD) at 25m footprint resolution, acquired via NASA Harmony API
* **NASA GEDI L2A v2** - canopy height metrics (rh98) acquired in monthly batches to avoid subsetter timeouts
* **ESA Sentinel-2 L2A** - cloud-masked median composites (Jan-Apr) acquired via OpenEO / Copernicus Data Space

### Pipeline
GEDI L4A + L2A  --+

(Harmony API)     +---> Feature Fusion ---> Gradient Boosting ---> Wall-to-Wall AGBD

Sentinel-2 L2A  --+    (GEDI x S2)         (rh98 + indices)       ---> Carbon Stock

(OpenEO)                                                            ---> Change Detection


**Step 1 - GEDI Acquisition & Preprocessing**
L4A (AGBD) and L2A (canopy height) downloaded as subsetted HDF5 files. L4A acquired as a single annual request; L2A in monthly batches. Beams parsed across all 8 GEDI beam groups. Shots joined on rounded lat/lon coordinates and filtered on: `l4_quality_flag == 1`, PFT class 1-5 (forest/shrub), AGBD 0-600 Mg/ha.

**Step 2 - Sentinel-2 Acquisition & Index Computation**
Cloud masking via Scene Classification Layer (SCL): vegetation (class 4) and bare soil (class 5) pixels retained. Median composite collapses the time dimension. Five spectral indices computed from scaled reflectance:

| Index | Formula | Sensitivity |
| --- | --- | --- |
| NDVI | (NIR - Red) / (NIR + Red) | Vegetation density |
| EVI | 2.5 x (NIR - Red) / (NIR + 6xRed - 7.5xBlue + 1) | Canopy structure |
| NDMI | (NIR - SWIR1) / (NIR + SWIR1) | Moisture content |
| NBR | (NIR - SWIR2) / (NIR + SWIR2) | Burn/disturbance |
| SAVI | 1.5 x (NIR - Red) / (NIR + Red + 0.5) | Soil-adjusted vegetation |

**Step 3 - Feature Fusion**
S2 feature values sampled at each GEDI footprint centre. Resolution mismatch noted: GEDI footprints are 25m diameter; S2 pixels are 10m. Single-pixel sampling is a documented simplification, reasonable given S2 contributes roughly 20% of model signal.

**Step 4 - Model Training & Benchmarking**
Three models benchmarked across three feature sets via 5-fold cross-validation:

| Feature Set | Gradient Boosting R² | Random Forest R² | Linear Regression R² |
| --- | --- | --- | --- |
| rh98 + S2 | **0.791** | 0.777 | 0.771 |
| rh98 only | 0.745 | 0.664 | 0.748 |
| S2 only | 0.043 | -0.013 | 0.003 |

S2-only R² near zero is consistent with **optical saturation** in dense tropical forest (AGBD > 150 Mg/ha), where spectral indices plateau and lose discriminative power. rh98 canopy height accounts for **79.3% of feature importance**. Sentinel-2 functions as spatial scaffolding enabling wall-to-wall extrapolation between sparse GEDI tracks, distinct from acting as a primary biomass predictor.

**Step 5 - Wall-to-Wall Prediction**
The 2020 model applied to every S2 pixel in both years. rh98 held at the year-specific GEDI mean (2020: 27.7m, 2022: 27.5m). Forest mask applied: NIR > 0.1, NDVI > 0.5, Blue < 0.15. A single model across both years keeps observed change attributable to biomass change rather than model variation between years.

**Step 6 - Carbon Estimation**
IPCC 2006 conversion factors applied:

Carbon stock (tC/ha) = AGBD x 0.47

CO2e (tCO2e/ha)      = Carbon stock x (44/12)

Change detection is restricted to pixels classified as forest in **both** years, to keep the comparison clear of cloud masking artefacts. The result is a net carbon decline over the period: gross emissions (3,876,060 tCO2e) outweigh gross sequestration (3,001,916 tCO2e).

---

## Key Findings

**Optical saturation is a demonstrated constraint, central to the methodology rather than a footnote.** Sentinel-2 indices alone achieve R² near 0.04 for AGBD in this forest type, with discriminative power dropping off above roughly 150 Mg/ha. This is the empirical basis for relying on GEDI LiDAR as the primary signal, with rh98 carrying 79.3% of feature importance.

**Forest carbon stock shows a measurable decline between 2020 and 2022.** The wall-to-wall raster comparison estimates a net change of -874,143 tCO2e, with gross emissions outweighing gross sequestration across the comparable forest area.

**The scope of this work is an exploratory feasibility study.** Its purpose is to translate ecological change into economic terms using Verra's frameworks as a structural reference, distinct in scope from a registered Verra project, baseline, or third-party verified MRV report.

---

## Limitations

This study operates within the following defined scope:

* **rh98 is held at the year-mean for wall-to-wall prediction.** Canopy height is not spatially interpolated across the island; the wall-to-wall AGBD map draws spatial texture from Sentinel-2 while rh98 is treated as locally uniform within each year.
* **Sentinel-2 sampling uses a single pixel at each GEDI footprint centre.** GEDI footprints are 25m; S2 pixels are 10m. This is a documented simplification, reasonable given S2's roughly 20% contribution to model signal.
* **Belowground biomass sits outside the current measurement scope.** It typically adds another 20-25% to total carbon stock and is reserved for future extension.
* **GEDI coverage varies between years.** 2022 cloud cover reduced L2A coverage to 4,297 shots versus 7,722 in 2020, narrowing spatial coverage available for the change estimate, with identical acquisition and filtering applied across both years.
* **The mean delta AGBD of -7.53 Mg/ha is read alongside the model's 22% RMSE.** The decline signal sits close to the model's own error margin and may partly reflect differences in 2022 cloud masking alongside real biomass change, a distinction the longitudinal extension is designed to clarify.
* **A counterfactual baseline, leakage assessment, and permanence analysis sit outside the current scope**, consistent with this being a feasibility study rather than a full REDD+ project design document.

---

## Future Updates

This README currently covers the 2020 vs 2022 phase of the project. Work is underway to extend the analysis into a longitudinal time series covering 2020, 2022, 2023, and 2025 (2021 sits outside scope due to corrupt L2A files; 2024 sits outside scope due to a GEDI instrument gap).

Planned additions:
* Longitudinal AGBD and rh98 trends across all four periods, building on the single-period comparison above
* A co-located footprint-based estimate of carbon stock change, complementing the wall-to-wall raster approach
* Bootstrapped confidence intervals around the change estimate
* Investigation into whether a canopy height decline alongside stable or rising NDVI holds across the full time series, a pattern consistent with selective logging rather than wholesale deforestation, and one invisible to optical-only monitoring

These will be published as an extension to this repository once finalized.

---

## Methodology References

| Framework | Purpose |
| --- | --- |
| Verra VM0042 | REDD+ crediting methodology |
| Verra VM0055 | Remote sensing MRV |
| Verra VT0005 | ALFB estimation via remote sensing |
| IPCC 2006 Guidelines | Carbon conversion factors |

---

## Repo Structure
+-- figures/
|   +-- agbd_map_2020.png
|   +-- agbd_map_2022.png
|   +-- feature_correlation.png
|   +-- feature_relationships.png
|   +-- gedi_tracks.png
|   +-- s2_bands_2020.png
|   +-- s2_bands_2022.png
+-- output/
|   +-- model_benchmark.csv
|   +-- model_dataset_2020.csv
|   +-- model_dataset_2022.csv
|   +-- project_summary.json
+-- Little_Andaman_Carbon_Stock_project.ipynb
+-- requirements.txt
+-- README.md
+-- .gitignore

> **Data files** (`.tif`, `.h5`, `.gpkg`, `.joblib`) are excluded via `.gitignore`, given GitHub file size limits. They can be reproduced by running the acquisition cells with valid NASA Earthdata and Copernicus Data Space credentials.

---
## Setup

git clone https://github.com/simran-pal/Little-Andaman-Carbon-Credit-Estimation.git

cd Little-Andaman-Carbon-Credit-Estimation
conda create -n gedi python=3.11

conda activate gedi

pip install -r requirements.txt


Credentials required:

* **NASA Earthdata** account for GEDI via Harmony API
* **Copernicus Data Space** account for Sentinel-2 via OpenEO

---

## Stack

Python | rasterio | geopandas | scikit-learn | h5py | OpenEO | NASA Harmony API | EPSG:32646

The non-obvious parts: parsing multi-beam GEDI HDF5 files, handling Harmony API timeouts with monthly L2A batches, and debugging a cloud mask inversion that was silently zeroing out valid pixels for weeks.
