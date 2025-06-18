## What does this project do?
This project was an exploration of new libraries and data processing techniques for me during an internship at Geosyntec Consultants, intended to show an example of what can be done with cloud processing and parallelization in handling remote sensing data. The basic idea is: 
- Get cloud-filtered satellite imagery from the summer seasons of a number of years, say 2018, 2019, 2020 and 2021.
- Train a regression model to predict wetland extents, based on Naturvårdsverkets own wetland inventory data (see https://www.naturvardsverket.se/publikationer/7100/978-91-620-7127-1/) and our downloaded 2018 satellite imagery as explanatory variable.
- Evaluate performance of the model on a subset of the 2018 data (80/20 training/validation split)
- Apply model across subsequent years satellite imagery to predict change in wetlands distribution over time!

## Prerequisites

### System Requirements
- Python 3.11 or higher
- Minimum 16GB RAM (recommended for large datasets)
- Sufficient storage space for satellite imagery and processed data 

### Python Dependencies

The project requires the following key Python packages (I strongly recommend creating a new, clean Conda env for use with Coiled):

```python
# Core scientific computing
numpy
xarray
dask
dask.distributed

# Remote sensing and geospatial
stackstac
pystac_client
planetary_computer
rioxarray
xrspatial

# Machine learning
scikit-learn
joblib

# Visualization and plotting
matplotlib

# Optional: Cloud computing
coiled  # For cloud-based processing
```

## Installation

1. Clone the repository:
```bash
git clone <repository-url>
cd Clean-Project-Folder
```

2. Create a virtual environment:
```bash
python -m venv my_clean_env
source my_clean_env/bin/activate  # On Windows: my_clean_env\Scripts\activate
```

3. Install dependencies:
```bash
conda install -c conda-forge numpy stackstac pystac_client planetary_computer xrspatial dask dask.distributed xarray bottleneck matplotlib rioxarray scikit-learn joblib
```

4. For cloud processing (optional):
```bash
conda install -c conda-forge coiled
```

## Usage

### Step 1: Data Acquisition and Processing

Run the `get_multiple_timestamps.ipynb` notebook to acquire and process Sentinel-2 satellite data:

1. **Configure Parameters**: Modify the configuration section to set:
   - Study area (GeoJSON file path)
   - Time periods (years and months)
   - Spatial resolution (default: 10m)
   - Cloud cover threshold (default: 20%)
   - Processing mode (local or cloud)

2. **Data Processing**: The notebook will:
   - Search for Sentinel-2 Level 2A data in the Planetary Computer STAC catalog
   - Filter images by cloud cover and temporal criteria
   - Apply cloud masking using SCL band
   - Generate median composite images for each time period
   - Save processed data as GeoTIFF files

### Step 2: Model Training and Prediction

Run the `train_and_predict_regression_model.ipynb` notebook to train the classification model:

1. **Data Preparation**: The notebook loads:
   - Explanatory variables (processed satellite imagery)
   - Target variable (ground truth labels)
   - Performs spatial alignment and resampling

2. **Model Training**: 
   - Splits data into training and validation sets
   - Trains a Random Forest classifier
   - Evaluates model performance with accuracy metrics and confusion matrix
   - Saves the trained model

3. **Prediction Generation**:
   - Loads the trained model
   - Processes all available time periods
   - Generates prediction maps for each time period
   - Saves results as GeoTIFF files

## Configuration

### Key Parameters

#### Data Acquisition (`get_multiple_timestamps.ipynb`)
- `local`: Boolean flag for local vs cloud processing
- `resolution`: Spatial resolution in meters (default: 10)
- `bands`: Sentinel-2 bands to process (default: ['B04', 'B03', 'B02', 'B07', 'B08', 'SCL'])
- `months`: Target months for analysis (default: ['june', 'july', 'august'])
- `years`: Target years for analysis (default: ['2018', '2019', '2020', '2021', '2022', '2023'])
- `tile_max_cloud`: Maximum cloud cover percentage per tile (default: 20)
- `max_items`: Maximum number of STAC items to process (default: 10)

#### Model Training (`train_and_predict_regression_model.ipynb`)
- `test_size`: Validation set proportion (default: 0.5)
- `n_estimators`: Number of Random Forest trees (default: 100)
- `max_samples`: Maximum samples per tree for memory management (default: 100000)

### Study Area Configuration

The study area is defined by a GeoJSON file located at `working_dir/study_area/Arvidsjaur.geojson`. This file should contain:
- A valid GeoJSON FeatureCollection or Polygon geometry
- Coordinates in a supported coordinate reference system
- Appropriate spatial extent for your analysis

## Data Sources

### Satellite Data
- **Sentinel-2 Level 2A**: Multi-spectral satellite imagery from the European Space Agency
- **Access**: Microsoft Planetary Computer STAC API
- **Spatial Resolution**: 10m (resampled from native 10-20m bands)
- **Temporal Coverage**: 2018-2023 (configurable)
- **Spectral Bands**: Blue (B02), Green (B03), Red (B04), NIR (B08), SWIR (B07), Scene Classification Layer (SCL)

### Ground Truth Data
- Training labels should be placed in `working_dir/labels/`
- Expected format: GeoTIFF with binary classification (0/1)
- Spatial resolution: 10m (matching satellite data)
- Coordinate reference system: EPSG:3006 (Swedish National Grid)
- This project originally used wetlands classification from Naturvårdsverket (https://www.naturvardsverket.se/publikationer/7100/978-91-620-7127-1/) as ground truth, converted from classified types of wetlands into wetland 0/1 dataset using ArcGIS Pro.

## Output Files

### Processed Satellite Data
- Location: `working_dir/`
- Format: GeoTIFF files
- Naming convention: `median_pixels_YYYY-MM-DD_YYYY-MM-DD.tif`
- Content: Multi-band median composite images

### Prediction Maps
- Location: `working_dir/prediction_maps/`
- Format: GeoTIFF files
- Naming convention: `pred_median_pixels_YYYY-MM-DD_YYYY-MM-DD.tif`
- Content: Binary classification maps (0: non-wetland, 1: wetland)

### Model Files
- Location: Project root directory
- Format: Joblib pickle file
- Filename: `random_forest_classifier.joblib`
- Content: Trained Random Forest classifier

## Performance Considerations

### Memory Management
- The workflow includes explicit memory cleanup to handle large datasets
- `max_samples` parameter in Random Forest controls memory usage
- Consider processing smaller areas or time periods for limited RAM systems

### Scalability
- Supports parallel processing using Dask
- Cloud-based processing available through Coiled
- Configurable chunk sizes, make sure to change this if using longer time series, higher/lower spatial resolution. A good chunksize is likely around 100MB, see Dask documentation.

### Future opportunities
- Implement a regression model based on Dask and XGBoost instead of Scikit-Learn, to be able to train on substantially more data in a parallelized way.
- Move from median of many images (compute inefficient) to instead use the first non-cloudy pixel among the stack of available pixels.

## Troubleshooting

### Common Issues

1. **Memory Errors**: Reduce `max_samples` or process smaller areas. A further development which would make training and prediction of larger datasets viable (which is highly desired from a model performance standpoint) is using XGBoost backed by Dask for parallelized and distributed training of the regression model.
2. **STAC API Timeouts**: Increase retry parameters or reduce `max_items`. This is an issue with this script - longer processing times in fetching data from the STAC (more than 30min) tend to time out and needs to have error handling added.
