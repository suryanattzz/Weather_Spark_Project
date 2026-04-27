# Weather Spark Project

This repository contains a Spark-based weather data pipeline built around the medallion architecture:

- **Bronze**: ingests raw weather CSV files into a Delta table
- **Silver**: cleans, validates, and standardizes the data
- **Gold**: builds analytical tables and aggregates for reporting

The project processes weather observations from the files in `raw_data/`, including `weather_01.csv` through `weather_13.csv`, plus the station reference file `station_master.csv`.

## Project Structure

- `notebook/bronze_notebook_weather.ipynb` - raw ingestion and bronze layer loading
- `notebook/silver_notebook_weather.ipynb` - data cleaning and transformation to silver tables
- `notebook/gold_notebook_weather.ipynb` - business-ready gold tables and analytics
- `raw_data/` - source CSV files used by the pipeline
- `screenshots/` - notebook output screenshots and visual references

## What the Pipeline Does

The notebooks appear to use a Databricks-style setup with the `weather_catalog` catalog and `bronze`, `silver`, and `gold` schemas. The bronze notebook reads CSV files from volume paths, writes Delta tables, and tracks ingestion metadata such as source file, batch id, and load date. The silver and gold notebooks then build refined datasets and summary tables for weather analysis.

## How to Run

1. Open the notebooks in a Spark environment such as Databricks.
2. Run the bronze notebook first to ingest the raw CSV files.
3. Run the silver notebook after the bronze layer is populated.
4. Run the gold notebook to generate the final analytical tables.
5. Review the generated outputs and screenshots in the `screenshots/` folder.

## Notes

- Update file paths or catalog names if your environment differs from the current notebook configuration.
- The notebooks were designed for Spark/Delta workflows and may require Databricks features such as volumes, catalog schemas, and Delta tables.
