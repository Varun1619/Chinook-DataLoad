# Chinook Data Pipeline

End-to-end metadata-driven data engineering pipeline built on the Chinook music store dataset, following the **Medallion Architecture** (Raw → Bronze → Silver → Gold) on Databricks.

**Course:** INFO 7374 — Data Architecture & Business Intelligence  
**Institution:** Northeastern University · Spring 2026

---

## Architecture

```
Source (Azure SQL)
        │
        ▼
┌──────────────┐
│   Raw Zone   │  Parquet files in Databricks Volume
│  (Volume)    │  /Volumes/workspace/raw_zone/chinook/{table}/{yyyy}/{mm}/{dd}/
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Bronze    │  Delta tables — exact copy of Raw, overwrite each run
│  (Delta)     │  Schema: workspace.bronze
└──────┬───────┘
       │  DQX Profiling + Validation
       ▼
┌──────────────┐
│    Silver    │  Delta tables — cleaned, nulls handled, failed records quarantined
│  (Delta)     │  Schema: workspace.silver
└──────┬───────┘
       │  Dimensional Modeling
       ▼
┌──────────────┐
│     Gold     │  Star schema — 7 dims (SCD1/SCD2) + 2 fact tables
│  (Delta)     │  Schema: workspace.gold
└──────────────┘
```

---

## Overview

This pipeline extracts data from an Azure SQL source, lands it in a Raw Volume as Parquet, promotes it through Bronze and Silver Delta tables with DQX data quality validation, and finally builds a star schema Gold layer with dimensional modeling and SCD Type 2 on `dim_customer`.

The pipeline is fully **metadata-driven** — a control table drives which tables run, and an execution log records every run's row counts and status. Everything is orchestrated via a Databricks Job with no hardcoded values in any notebook.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks + Unity Catalog | Compute, governance, Delta table management |
| Delta Lake | ACID transactions, SCD Type 2 merge |
| Databricks DQX | Data quality profiling and validation |
| Databricks Workflows | End-to-end job orchestration |
| PySpark | All transformation logic |
| Azure SQL (Chinook) | Source database |

---

## Pipeline Notebooks

| Notebook | What it does |
|---|---|
| `01_extract_from_source` | Reads `pipeline_control`, loops active tables, extracts via Connection Manager |
| `02_load_raw` | Writes Parquet to Volume with dated paths, logs to `execution_log` |
| `03_raw_to_bronze` | Reads latest successful path from `execution_log`, writes Delta to Bronze (overwrite) |
| `04_bronze_to_silver` | DQX profiling (nulls, dupes, ranges), validates rules, quarantines failures, writes Silver |
| `05_silver_to_gold` | Builds 7 dims (SCD Type 1) + `dim_customer` (SCD Type 2) + `fact_sales` + `fact_sales_customer_agg` |

---

## Gold Layer

| Table | Type | Source |
|---|---|---|
| `dim_artist` | Dimension (SCD1) | silver.artist |
| `dim_album` | Dimension (SCD1) | silver.album |
| `dim_genre` | Dimension (SCD1) | silver.genre |
| `dim_media_type` | Dimension (SCD1) | silver.mediatype |
| `dim_employee` | Dimension (SCD1) | silver.employee |
| `dim_track` | Dimension (SCD1) | silver.track |
| `dim_customer` | Dimension **(SCD2)** | silver.customer |
| `fact_sales` | Fact | silver.invoiceline + silver.invoice + dim_customer |
| `fact_sales_customer_agg` | Aggregate Fact | gold.fact_sales |

---

## How to Run

1. Configure source in Databricks Connection Manager as `chinook_connection_catalog`
2. Create schemas `bronze`, `silver`, `gold` under `workspace` catalog
3. Create Volume: `workspace.raw_zone.chinook`
4. Run the metadata table setup SQL (`pipeline_control` + `execution_log`)
5. Create a Databricks Job named `Chinook_Pipeline` with all 5 notebooks in order
6. Set Job-level parameters:

```
catalog_name  = workspace
schema_name   = bronze
base_path     = /Volumes/workspace/raw_zone/chinook
silver_schema = silver
table_name    = all
```

7. Click **Run now** — verify results in `execution_log` after completion

---

## Contributors

| Name | Role | Links |
|---|---|---|
| Varun | Gold Layer, Mapping Document, Job Orchestration | Northeastern University |
| Krupali Tejani | Raw → Bronze → Silver, DQX Validation | [GitHub](https://github.com/Krupali04) |
| Akshay Govind | Environment Setup, Metadata Tables, Extract Notebook | [GitHub](https://github.com/akshaygovindd) · [Email](mailto:govind.ak@northeastern.edu) |
