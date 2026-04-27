# Databricks Weather Analytics Project

This repository implements an end-to-end weather data pipeline on Databricks using Medallion architecture:

- Bronze: raw archival ingestion using Auto Loader and Delta
- Silver: CDF-driven incremental cleaning, quarantine, and MERGE-based upserts
- Gold: business-facing aggregates for monitoring and analytics

Cities in scope:

- Chennai
- Bangalore
- Hyderabad
- Mumbai
- Delhi
- Pune

## Tech Stack

| Component | Tool / Feature |
|---|---|
| File ingestion | Auto Loader (`cloudFiles`) |
| Storage format | Delta Lake |
| Incremental processing | Change Data Feed (CDF) |
| Recovery and history | Delta Time Travel |
| Orchestration | Databricks Workflows |
| Language | PySpark / Spark SQL |
| Catalog | Unity Catalog |

## Repository Layout

- `notebook/bronze_notebook_weather.ipynb` -> Stage 1 Bronze ingestion
- `notebook/silver_notebook_weather.ipynb` -> Stage 2 Silver transformation
- `notebook/gold_notebook_weather.ipynb` -> Stage 3 Gold aggregation
- `raw_data/` -> 13 weather CSV files + station reference CSV
- `screenshots/` -> validation screenshots and demo output

## Databricks Storage Layout (Target)

Create these paths before ingestion:

- `/weather-project/data/raw` -> landing zone for all 13 weather files
- `/weather-project/data/reference` -> station reference file (`station_master.csv`)
- `/weather-project/checkpoints/` -> Auto Loader checkpoints (never delete)
- `/weather-project/configs/` -> schema and configuration assets

Current notebook implementation uses Unity Catalog volumes under `weather_catalog`, for example:

- `/Volumes/weather_catalog/data/raw_files/`
- `/Volumes/weather_catalog/data/reference_file/`
- `/Volumes/weather_catalog/data/schema`
- `/Volumes/weather_catalog/bronze/checkpoint`

## Dataset Plan

The ingestion model is file-based incremental processing:

- Drop all weather files into raw landing zone up front.
- Auto Loader runs with `maxFilesPerTrigger = 1`.
- Each run processes one new file and stops (`trigger(availableNow=True)`).
- Checkpoint state prevents duplicate ingestion of already-seen files.

Input files:

- Weather data: `weather_01.csv` through `weather_13.csv`
- Reference data: `station_master.csv` (12 station IDs)

## Stage 1 - Bronze Ingestion

Notebook: `notebook/bronze_notebook_weather.ipynb`

### Intended delivery (spec)

- `bronze_weather`
- `station_master` (one-time batch load, static)
- notebook name reference: `01_bronze_ingestion.py`

### Implemented in current notebook

- Creates catalog, schemas, and volumes in `weather_catalog`
- Uses Auto Loader with CSV source (`cloudFiles`) and file-based incremental trigger
- Uses `maxFilesPerTrigger = 1`
- Adds ingestion metadata columns:
	- `ingestion_time`
	- `source_file`
	- `batch_id`
	- `load_date`
	- `record_status`
- Writes Bronze Delta table in append mode with CDF enabled on first write
- Partitions by `load_date`

Current Bronze table name in notebook code:

- `weather_catalog.bronze.bronze_weather_tbl`

Important note:

- `record_status` is currently set to `valid` in notebook code; spec asks for `raw` for all Bronze rows.
- Schema evolution in active Bronze code is currently configured as `rescue`; spec asks for `addNewColumns` with `_rescued_data` capture.
- Reference station table load is expected by Silver notebook (`weather_catalog.bronze.ref_st_master_tbl`) but is not fully shown as a dedicated one-time batch load cell in the current Bronze notebook.

## Stage 2 - Silver Transformation

Notebook: `notebook/silver_notebook_weather.ipynb`

### Intended delivery (spec)

- `silver_weather_clean` (CDF enabled)
- `silver_weather_quarantine`
- `weather_change_audit`
- `pipeline_control`
- notebook name reference: `02_silver_transformation.py`

### Implemented in current notebook

- Creates tracking table (`weather_catalog.silver.version_tbl`) to store processed Bronze version
- Reads Bronze incrementally using CDF (`readChangeFeed=true`, `startingVersion` from tracking table)
- Filters CDF rows to keep `insert` and `update_postimage`
- Expands rescued JSON content from `_rescued_data` and `sensor_payload`
- Handles provider column variation with `coalesce(weather_condition, condition)`
- Applies `try_cast`-based type normalization for canonical fields
- Validates and quarantines rows with reason codes:
	- `invalid_timestamp`
	- `out_of_range_wind_speed`
	- `out_of_range_temperature`
	- `out_of_range_humidity`
	- `station_not_in_master`
- Adds derived columns:
	- `event_date`
	- `event_hour`
	- `temperature_band`
	- `weather_severity_score`
- Uses `MERGE INTO` on `(city, event_time)` and ingestion-time tie-breaker
- Updates processed version table after successful write
- Writes quarantine table as Delta append
- Writes an audit table by comparing previous and latest Silver versions

Current Silver table names in notebook code:

- `weather_catalog.silver.silver_weather_clean`
- `weather_catalog.silver.silver_weather_quarantine`
- `weather_catalog.silver.weather_audit_tbl`
- `weather_catalog.silver.version_tbl`


## Stage 3 - Gold Aggregation

Notebook: `notebook/gold_notebook_weather.ipynb`

### Intended delivery (spec)

- `gold_city_weather_snapshot`
- `gold_heatwave_alerts`
- `gold_rainfall_streaks`
- `gold_temperature_change`
- `gold_provider_quality`
- notebook name reference: `03_gold_aggregation.py`

Current Gold table names in notebook code:

- `weather_catalog.gold.gold_cur_city_weather`
- `weather_catalog.gold.gold_cur_city_temp_occurrence`
- `weather_catalog.gold.gold_city_rain_streak`
- `weather_catalog.gold.gold_temperature_change`
- `weather_catalog.gold.gold_provider_quality`


## Stage 4 - Workflow and Orchestration (Databricks Jobs)

Recommended Databricks Job:

- Job name: `weather_analytics_pipeline`
- Task 1: Bronze notebook
- Task 2: Silver notebook (depends on Task 1)
- Task 3: Gold notebook (depends on Task 2)
- Task 4: Data quality notebook (optional, depends on Task 3)
- Schedule: hourly
- Failure notifications: enabled for team email

## Run Order

1. Upload all weather files (`weather_01.csv` to `weather_13.csv`) to raw landing zone.
2. Upload `station_master.csv` to reference landing zone and load station reference Delta table.
3. Run Bronze notebook.
4. Run Silver notebook.
5. Run Gold notebook.

## Stage-to-Notebook Naming Map

Specification notebook names vs repository notebook files:

- `01_bronze_ingestion.py` -> `notebook/bronze_notebook_weather.ipynb`
- `02_silver_transformation.py` -> `notebook/silver_notebook_weather.ipynb`
- `03_gold_aggregation.py` -> `notebook/gold_notebook_weather.ipynb`


