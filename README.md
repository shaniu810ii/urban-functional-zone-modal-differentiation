# Urban Functional-Zone Modal Differentiation

This repository provides the released code and partial data used for the study **Nonlinear Transport Modal Differentiation across Urban Functional Zones: A Data-Driven Study with Generalized Additive Models**. The workflow covers POI-based urban functional-zone identification, multimodal OD processing, enrichment-factor calculation, source-sink analysis, and distance-time interaction GAM modeling.

Large files in the `Data/` directory are managed with Git LFS. Please install Git LFS before cloning or downloading the repository with Git.

## Repository Contents

```text
.
|-- Code/
|-- Data/
|   |-- POI/
|   |-- Region/
|   |-- Road/
|   |-- share_bike/
|   |-- subway_bus/
|   `-- taxi/
|-- .gitattributes
`-- README.md
```

## Code Files

| File | Description |
|---|---|
| `Code/0.POI_preproces.txt` | Preprocesses raw POI shapefiles and prepares cleaned POI layers for subsequent sequence construction. |
| `Code/1.POI_seq.txt` | Constructs POI category sequences around road-network sampling points. |
| `Code/2.Doc2Vec.txt` | Trains Doc2Vec representations from POI sequences and exports community-level vectors. |
| `Code/3.K-means.txt` | Clusters community vectors and supports functional-zone classification. |
| `Code/4.EF.txt` | Calculates enrichment factors for interpreting and naming functional zones. |
| `Code/5.distance_duration_.txt` | Summarizes travel-distance and travel-duration distributions for multiple transport modes. |
| `Code/6.Time×Function_pair.txt` | Builds time-period functional-zone OD matrices and heatmap-ready outputs. |
| `Code/7.source_sink.txt` | Calculates source-sink indicators by functional zone, transport mode, and time period. |
| `Code/8.GAM.R` | Fits and visualizes distance-time interaction GAMs for modal probability analysis. |
| `Code/9.Bike_data_processing.txt` | Processes sample bike-sharing trip records into analysis-ready OD data. |
| `Code/10.Bus_data_processing.txt` | Processes sample bus trip records into analysis-ready OD data. |
| `Code/11.Subway_data_processing.txt` | Processes sample subway trip records into analysis-ready OD data. |
| `Code/12.Taxi_data_processing.txt` | Processes sample taxi trip records into analysis-ready OD data. |

The code files use relative paths and avoid personal local file paths. The `.txt` files contain the released script contents and can be adapted to local Python workflows if needed.

## Data Files

The `Data/` directory contains partial or supporting data for code inspection and reproducibility checks:

| Folder | Description |
|---|---|
| `Data/POI/` | POI shapefile components used in the functional-zone identification workflow. |
| `Data/Region/` | Community or regional boundary shapefile components. |
| `Data/Road/` | Road-network shapefile components used for POI sequence construction. |
| `Data/share_bike/` | Sample bike-sharing trip data files. |
| `Data/subway_bus/` | Split subway-bus sample data files. The released sample contains the first 50% of the source table, divided into 45 text files. |
| `Data/taxi/` | Sample taxi trip data files. |

All files under `Data/` are tracked with Git LFS because several files are large. After cloning, run:

```text
git lfs install
git lfs pull
```

## Suggested Workflow

The main analytical scripts are organized in the following order:

```text
0.POI_preproces.txt
1.POI_seq.txt
2.Doc2Vec.txt
3.K-means.txt
4.EF.txt
5.distance_duration_.txt
6.Time×Function_pair.txt
7.source_sink.txt
8.GAM.R
```

The transport-mode preprocessing scripts can be used before the OD analysis scripts when starting from the released sample trip files:

```text
9.Bike_data_processing.txt
10.Bus_data_processing.txt
11.Subway_data_processing.txt
12.Taxi_data_processing.txt
```

## Environment

The Python scripts require common geospatial and data-science packages, including:

```text
geopandas
pandas
numpy
shapely
gensim
scikit-learn
matplotlib
seaborn
openpyxl
```

The R script requires:

```text
data.table
mgcv
ggplot2
```

The spatial scripts use projected coordinate reference system `EPSG:4527`.

## Notes

This repository is intended to support review, reuse, and partial reproduction of the analytical workflow. Some source datasets may be represented by samples or supporting files rather than the full original raw records.
