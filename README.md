# Nuremberg Land-Cover Intelligence

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB.svg)](https://www.python.org/)
[![LightGBM](https://img.shields.io/badge/Model-LightGBM-2C7A7B.svg)](https://lightgbm.readthedocs.io/)
[![Streamlit](https://img.shields.io/badge/App-Streamlit-FF4B4B.svg)](https://streamlit.io/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A tabular geospatial machine-learning pipeline for estimating land-cover composition and change across Nuremberg on a 100 m grid. It combines Sentinel-2 surface reflectance, spectral indices and ESA WorldCover supervision with spatial cross-validation and an interactive analysis application.

## What it predicts

For every grid cell and year from 2019 to 2023, the pipeline estimates the proportion of:

- built-up area
- vegetation
- water
- other land cover

It also derives year-to-year changes and uncertainty summaries.

## Validation

Five spatial folds were used to reduce overly optimistic scores caused by geographic autocorrelation.

| Built-up proportion metric | Cross-validation result |
|---|---:|
| MAE | **0.0489 ± 0.0029** |
| RMSE | **0.0979 ± 0.0050** |
| R² | **0.9146 ± 0.0091** |

## Pipeline

~~~mermaid
flowchart TD
    A["Sentinel-2 composites"] --> C["100 m grid features"]
    B["ESA WorldCover labels"] --> D["Training table"]
    C --> D
    D --> E["Spatial CV and LightGBM"]
    E --> F["Yearly composition"]
    F --> G["Change and uncertainty"]
    G --> H["Streamlit application"]
~~~

## Features

- Sentinel-2 B2, B3, B4 and B8 reflectance
- NDVI and NDWI
- 100 m grid in EPSG:25832
- LightGBM ensemble with Ridge baseline
- Spatial cross-validation and Optuna tuning
- Interactive AOI selection using rectangles or polygons
- Composition, change, confidence and population views
- Cell-level temporal inspection and CSV/PDF export

## Quick start

~~~bash
git clone https://github.com/sandeep848/nuremberg-land-cover-ml.git
cd nuremberg-land-cover-ml
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
streamlit run app/streamlit_app.py
~~~

On Windows, activate with `.venv\Scripts\activate`.

Processed predictions are required for the application. When they are unavailable, run the preparation pipeline below.

## Training pipeline

~~~bash
python src/make_grid.py \
  --raster data/raw/sentinel/S2_2020_Jun01_Aug31_10m_QA60SCL_F32.tif \
  --cell-size 100 \
  --output data/processed/grid/grid_100m.gpkg

python src/build_training_table.py \
  --grid data/processed/grid/grid_100m.gpkg \
  --features-dir data/processed/features \
  --labels-dir data/processed/labels \
  --years 2020 2021 \
  --output data/processed/tables/train_table.parquet

python src/train_models.py \
  --train data/processed/tables/train_table.parquet \
  --outdir models \
  --spatial-folds 5 \
  --optuna-trials 30 \
  --ensemble-size 5

python src/predict_all_years.py \
  --features-dir data/processed/features \
  --years 2019 2020 2021 2022 2023 \
  --model-dir models \
  --output-dir data/processed/predictions \
  --include-uncertainty
~~~

Use `src/extract_features.py` for each yearly Sentinel composite and `src/extract_labels.py` for the 2020 and 2021 WorldCover rasters before building the training table.

## Repository structure

~~~text
src/                    # Grid, feature, label, training and prediction scripts
app/streamlit_app.py    # Interactive application
data/raw/               # Source rasters
data/processed/         # Derived tables and predictions
models/                 # Models and validation summaries
config.yaml             # Project configuration
~~~

## Data and reproducibility

Raw Sentinel-2 and WorldCover data are not redistributed by this project. Generated model and geospatial artifacts currently included in the repository are intended to make the demonstrated analysis reproducible; larger artifacts should eventually move to a versioned release or external data registry.

## Limitations

- WorldCover provides supervision for 2020 and 2021, so estimates outside those years rely on temporal transfer.
- Predictions describe grid-cell composition rather than individual 10 m pixels.
- Spatial cross-validation reduces, but does not eliminate, geographic dependence.
- Population overlays provide context and are not used as causal evidence of land-cover change.

## License

Distributed under the [MIT License](LICENSE).
