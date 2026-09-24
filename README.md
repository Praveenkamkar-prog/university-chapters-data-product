# University Chapters Data Product

## Overview

This project implements a thin Medallion data pipeline for the
University Chapters dataset.

The pipeline extracts data from a public ArcGIS REST API and processes it
through Bronze, Silver and Gold layers using Databricks and PySpark.

The Gold layer provides a clean, validated and consumer-ready data product
for university chapter analytics and reporting.

## Data Scope

The pipeline processes university chapter records for:

- California (CA)
- Oregon (OR)
- Washington (WA)

Oregon and Washington may legitimately contain zero records based on the
source data.

## Architecture

The pipeline follows this flow:
```text
ArcGIS REST API
        |
        v
     Bronze
        |
        v
     Silver
        |
        v
  Data Quality
     /     \
    /       \
Quarantine  Valid
              |
              v
            Gold

```
## Technologies

- Databricks
- Apache Spark / PySpark
- Python
- Unity Catalog
- Parquet
- ArcGIS REST API

## Project Structure

```text
university-chapters-data-product/
├── notebooks/
│   └── university_chapters_pipeline
├── docs/
│   ├── architecture.md
│   └── data_product_contract.md
└── README.md
```


## Prerequisites

The solution requires:

- A Databricks workspace
- Access to the project Git repository
- A Unity Catalog-enabled Databricks environment
- Spark / PySpark
- Internet access from the Databricks environment to the source API

## Unity Catalog Setup

The implementation uses:

- Catalog: university_chapters
- Schema: data_product
- Volume: project_files

The main project volume is:

/Volumes/university_chapters/data_product/project_files


## Output Paths

Bronze:

/Volumes/university_chapters/data_product/project_files/bronze/university_chapters/<run_id>

Silver:

/Volumes/university_chapters/data_product/project_files/silver/university_chapters

Gold:

/Volumes/university_chapters/data_product/project_files/gold/university_chapters/v1

Quarantine:

/Volumes/university_chapters/data_product/project_files/quarantine/university_chapters/<run_id>

Audit:

/Volumes/university_chapters/data_product/project_files/audit/university_chapters

## How to Run

1. Clone the GitHub repository into Databricks using Databricks Git folders.
2. Open the pipeline notebook under the `notebooks` folder.
3. Attach the notebook to a Databricks compute resource.
4. Confirm that the Unity Catalog catalog, schema, and volume are available.
5. Run the notebook from the beginning.
6. The notebook extracts CA, OR and WA records from the ArcGIS API.
7. The pipeline processes the data through Bronze, Silver, Quarantine and Gold.
8. Review the audit metrics and automated DQ test results at the end of the notebook.

## Data Quality Rules

### DQ-Q1 - Invalid Coordinates

This is a hard failure rule.

A record is quarantined when:

- Longitude is null
- Latitude is null
- Longitude is outside the range -180 to 180
- Latitude is outside the range -90 to 90

The record does not enter the valid Silver dataset or Gold.

Quarantine reason:

INVALID_COORDINATES

### DQ-W1 - Missing or Unknown City

This is a warning rule.

A record receives a warning when:

- City is null
- City is blank
- City is UNKNOWN, regardless of case

The record is still published to Gold.

The Gold record contains:

dq_status = WARNING

dq_warnings = MISSING_OR_UNKNOWN_CITY

Clean records contain:

dq_status = OK

dq_warnings = empty

## Testing

The notebook includes fixture-driven automated tests for the data quality rules.

The test dataset contains:

- One record with invalid coordinates
- One record with a missing city
- One clean record

Expected results:

- Invalid-coordinate record is excluded from valid Silver and Gold.
- Missing-city record is published to Gold with WARNING status.
- Clean record is published to Gold with OK status.

The notebook uses assertions to automatically validate these expected outcomes.

## Audit Metrics

The pipeline records the following metrics for each ingestion run:

- rows_in
- rows_quarantined
- rows_warned
- rows_ok

These metrics provide visibility into the number of records processed and
the outcome of the data quality checks.

## Design Decisions

### Bronze

The raw API payload is retained in Bronze together with ingestion metadata.
Bronze data is stored by ingestion run to support traceability and
troubleshooting.

### Silver

Silver performs parsing, data type conversion, cleansing and deduplication
at the chapter_id grain.

### Gold

Gold contains the consumer-facing data product. Quarantined records are
excluded, while warning records remain available with their DQ status and
warning information.

### Idempotency

Bronze retains historical ingestion runs.

Silver and Gold use overwrite behavior for their current output paths so that
re-running the notebook does not continuously append duplicate records.

A production implementation could use Delta Lake and MERGE for incremental
idempotent processing.

### Take-home Scope

This implementation intentionally keeps the infrastructure simple.

Production extensions could include:

- ADLS Gen2
- Delta Lake
- ADF orchestration
- CI/CD
- Secret management
- Centralized monitoring
- Production data catalog and governance