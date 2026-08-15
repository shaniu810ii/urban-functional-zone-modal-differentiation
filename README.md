# Urban Functional-Zone Modal Differentiation

This repository provides the source code used for the study **Nonlinear Transport Modal Differentiation across Urban Functional Zones: A Data-Driven Study with Generalized Additive Models**. The workflow combines POI-based urban functional-zone identification, multimodal origin-destination analysis, source-sink analysis, and distance-time interaction GAM modeling.

## Overview

The code supports three main analytical tasks:

1. Identifying urban functional zones from POI semantic sequences.
2. Analyzing multimodal OD travel patterns among functional zones.
3. Quantifying nonlinear modal differentiation using distance-time interaction GAMs.

The released scripts use relative paths and do not include personal local file paths. Raw OD records, POI shapefiles, road-network files, and community-boundary data are not redistributed because they may be subject to data-use restrictions.

## Code Files

| File | Extended description |
|---|---|
| `0.POI_preproces.txt` | Optional POI preprocessing utility. This script reads raw POI shapefiles from `data/raw_poi_shp/`, filters records to the central Beijing study districts, parses hierarchical POI category information, removes non-analytical facility categories, and exports cleaned POI shapefiles to `data/poi_shp/`. It standardizes POI attributes before sequence construction and functional-zone analysis. |
| `1.POI_seq.txt` | POI sequence construction script. It generates regularly spaced sample points along the road network, creates local buffers around these points, spatially joins nearby POIs, ranks POIs by distance, and builds ordered POI category sequences for each sampling point. These sequences are used as textual-like inputs for Doc2Vec-based urban functional representation. |
| `2.Doc2Vec.txt` | Doc2Vec representation model script. It loads POI sequences, attaches community-unit identifiers to sampled road-network points, trains a Doc2Vec model, and exports community-level document vectors. These vectors capture the semantic composition of local urban functions based on surrounding POI categories. |
| `3.K-means.txt` | Functional-zone clustering script. It clusters Doc2Vec community vectors using K-means, evaluates alternative cluster numbers with the Davies-Bouldin Index, attaches cluster labels to community polygons, and exports both spatial clustering results and visualization figures. The six-cluster solution is used as the functional-zone framework in later analyses. |
| `4.EF.txt` | Enrichment-factor calculation script. It spatially joins POIs to community units, aggregates POI classes by K-means functional zone, and computes the relative overrepresentation of each POI category compared with the citywide distribution. The EF table supports interpretation and naming of the identified functional zones. |
| `5.distance_duration_.txt` | Travel-distance and travel-duration analysis script. It processes multimodal OD records for bike sharing, bus, subway, and taxi; bins travel distances and durations; and generates publication-ready distribution figures. These outputs describe basic modal differences in spatial and temporal travel ranges. |
| `6.Time×Function_pair.txt` | Functional-zone OD assignment and heatmap script. It assigns trip origins and destinations to functional-zone polygons, classifies departure time into daily periods, constructs OD matrices by mode and time period, and visualizes them as heatmaps. The output supports comparison of multimodal interactions among functional-zone pairs. |
| `7.source_sink.txt` | Source-sink analysis script. It calculates hourly net inflow rates for each functional zone and transport mode using origin and destination counts, then produces zone-specific curves and a summary figure. The results reveal temporal shifts in whether each functional zone acts mainly as a trip source or sink across the day. |
| `8.GAM.R` | Distance-time interaction GAM script. It provides reusable R functions for aggregating trip counts by distance and departure-time bins, fitting multinomial GAMs, predicting continuous modal probability curves, estimating uncertainty intervals, extracting threshold-like transition intervals, and plotting Figure 8-style modal probability curves. The script quantifies how modal dominance changes with travel distance, departure time, and functional OD context. |

## Expected Directory Structure

```text
.
├── data/
│   ├── raw_poi_shp/
│   ├── poi_shp/
│   ├── road_network/
│   ├── community_units/
│   ├── study_area_boundary/
│   └── od/
├── outputs/
│   └── figures/
└── code files
```

## Main Input Data

The scripts assume equivalent input data are placed under the `data/` directory:

- POI shapefiles
- Road-network shapefile
- Community-unit polygon shapefile
- Central study-area boundary shapefile
- Multimodal OD records with mode, distance, duration, origin point, destination point, and departure-hour information

The coordinate reference system used in the spatial scripts is `EPSG:4527`.

## Python Environment

The Python scripts require packages including:

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

## R Environment

The GAM script requires:

```text
data.table
mgcv
ggplot2
```

## Suggested Workflow

Run the scripts in the following order:

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

`8.GAM.R` is organized as a set of reusable functions rather than a single standalone pipeline. After preparing trip records with `od_pair`, `mode`, `distance_km`, and `time_hour`, users can call the aggregation, model-fitting, prediction, threshold-extraction, and plotting functions to reproduce the GAM-based modal probability analysis.

## Outputs

The workflow generates intermediate data and figures under `outputs/`, including:

- Cleaned POI tables and spatial joins
- POI sequence files
- Doc2Vec model and document vectors
- K-means clustering results
- EF tables
- Travel-distance and travel-duration figures
- Source-sink curves
- Functional-zone OD heatmaps
- GAM-based modal probability curves and threshold visualizations

## Notes

The repository contains code only. To reproduce the full analysis, users need to prepare equivalent input datasets using the directory structure above. File paths are intentionally relative so that the workflow can be adapted to different local environments.
