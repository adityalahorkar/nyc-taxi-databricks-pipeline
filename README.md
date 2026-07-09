# NYC Taxi Trip Analytics — Databricks Data Pipeline

An end-to-end data engineering pipeline built on Databricks that ingests, cleans,
and aggregates NYC TLC Yellow Taxi trip data using a medallion architecture
(Bronze/Silver/Gold), orchestrated via scheduled Databricks Workflows with
idempotent incremental loading, and visualized in Power BI.

## Architecture

Raw Parquet Files (NYC TLC)
│
▼
┌─────────────────┐
│  BRONZE LAYER    │  Raw ingestion, schema preserved, audit columns added
│  (Delta Table)   │  Incremental MERGE — safe to re-run without duplicates
└────────┬─────────┘
▼
┌─────────────────┐
│  SILVER LAYER    │  Data quality filtering, derived metrics, zone-name joins
│  (Delta Table)   │  Assertions on nulls/ranges before write
└────────┬─────────┘
▼
┌─────────────────┐
│  GOLD LAYER      │  Pre-aggregated tables (hourly demand, daily summary,
│  (Delta Tables)  │  borough flow, top zones) — optimized for BI queries
└────────┬─────────┘
▼
Databricks SQL Warehouse ──► Power BI Dashboard

Orchestrated end-to-end via a Databricks Workflow (3-task DAG), scheduled to
run automatically, with each new monthly data file processed incrementally.

## Key Engineering Decisions

- **Incremental loading with `MERGE INTO`** instead of full overwrites in
  Bronze and Silver layers — allows new monthly files to be added without
  reprocessing or duplicating historical data
- **Idempotent design** — re-running the pipeline on the same data produces
  no duplicate rows, verified by matching on `source_file` + composite keys
- **Unity Catalog compliant file tracking** — used `_metadata.file_path`
  instead of the deprecated `input_file_name()` function
- **Data quality gates** — explicit assertions on row counts, null rates,
  and value ranges before any data is written, so bad data fails the job
  rather than silently propagating downstream
- **Delta Lake optimization** — applied `OPTIMIZE` + `ZORDER` on frequently
  filtered columns (`PULocationID`, `pickup_date`) to improve query performance
- **Dimensional modeling** — separated the taxi zone lookup into its own
  dimension table and joined it in for both pickup and dropoff locations

## Data Quality Results

Bronze rows: 9,554,778
Silver rows: 8,473,894
Filtered out: 11.31%
Zone join null rate: 0.0%
Gold daily summary rows: 96
Gold hourly demand rows: 209980

## Gold Tables (Power BI-ready)

| Table | Purpose | Rows |
|---|---|---|
| `gold_hourly_demand` | Trip volume/revenue by hour + zone | [FILL IN] |
| `gold_daily_summary` | Daily KPIs for trend charts | [FILL IN] |
| `gold_borough_flow` | Pickup → dropoff borough matrix | [FILL IN] |
| `gold_top_zones` | Ranked busiest zones | [FILL IN] |

## Tech Stack

- **Databricks Free Edition** — serverless compute, notebooks, SQL Warehouse
- **PySpark** — data transformation and cleaning
- **Delta Lake** — ACID transactions, MERGE, OPTIMIZE/ZORDER
- **Unity Catalog** — catalog/schema governance, Volumes for file landing
- **Databricks Workflows** — job orchestration, scheduling, task dependencies
- **Power BI** — dashboarding via Databricks SQL Warehouse connector

## Dataset

NYC TLC Trip Record Data (official source):
https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

Yellow Taxi trip records, [FILL IN months used, e.g. "Jan–Mar 2024"],
plus the official Taxi Zone Lookup table.

## Pipeline Screenshots

See `/docs` folder for:
- Job DAG and task dependencies
- Scheduled run history
- Sample Gold table output
- Power BI dashboard (if included)

## Repository Structure

notebooks/
01_bronze_ingest.py     — raw ingestion with incremental MERGE
02_silver_clean.py      — cleaning, derived columns, zone joins, DQ checks
03_gold_aggregate.py    — aggregation into BI-ready Gold tables
docs/
*.png                   — screenshots of job runs, DAG, dashboard

## How to Reproduce

1. Sign up for Databricks Free Edition
2. Create catalog `nyc_taxi_project` with schema `raw` and a Volume for landing files
3. Download NYC TLC Yellow Taxi parquet files + zone lookup CSV, upload to the Volume
4. Import the 3 notebooks from `/notebooks` into your workspace
5. Create a Databricks Workflow chaining the 3 notebooks with dependencies
6. Run the job, then connect Power BI (or the Databricks SQL Editor) to the Gold tables
