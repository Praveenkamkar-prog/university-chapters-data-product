# University Chapters Data Product Contract

## 1. Data Product Overview

**Data Product Name:** University Chapters Data Product

**Owner:** Data Engineering

**Purpose:**  
Provide a clean, validated and consumer-ready dataset of university chapters
for the states of California (CA), Oregon (OR), and Washington (WA).

**Primary Use Cases:**
- Analytics and reporting
- University chapter analysis
- Location-based analysis
- Downstream data consumption

**Source:**  
ArcGIS UniversityChapters_Public API

**Data Scope:**  
Only CA, OR and WA records are included.

## 2. Gold Data Interface

**Gold Path:**

`/Volumes/university_chapters/data_product/project_files/gold/university_chapters/v1`

**Format:** Parquet

**Grain:** One row per `chapter_id`

### Gold Schema

| Column | Type | Nullable | Description |
|---|---|---|---|
| `chapter_id` | string | No | Unique university chapter identifier |
| `chapter_name` | string | Yes | University chapter name |
| `city` | string | Yes | City where the chapter is located |
| `state` | string | No | USPS 2-letter state code: CA, OR or WA |
| `longitude` | double | No | WGS84 longitude |
| `latitude` | double | No | WGS84 latitude |
| `dq_status` | string | No | `OK` or `WARNING` |
| `dq_warnings` | array<string> | No | Data-quality warning codes |

### Consumer Expectations

Consumers should use `chapter_id` as the chapter-level business key.

Quarantined records are not published to Gold.

Records with a city-quality warning are still published to Gold with
`dq_status = 'WARNING'` and the applicable warning code.

## 3. Data Quality Rules

### DQ-Q1 — Invalid Coordinates

**Severity:** Hard failure

A record fails DQ-Q1 when:

- `longitude` is null
- `latitude` is null
- `longitude` is outside the range `-180 to 180`
- `latitude` is outside the range `-90 to 90`

Failed records:

- Do not enter the valid Silver dataset
- Do not enter Gold
- Are written to the quarantine path
- Receive the reason code `INVALID_COORDINATES`
- Retain `ingest_run_id` and raw payload information for debugging

### DQ-W1 — Missing or Unknown City

**Severity:** Warning

A record receives a warning when `city` is:

- null
- blank
- `UNKNOWN` (case-insensitive)

The record is still published to Gold with:

```text
dq_status = WARNING
dq_warnings = ["MISSING_OR_UNKNOWN_CITY"]
dq_status = OK
dq_warnings = []
```

## 4. Batch and Freshness Rules

### State Scope

The pipeline processes only:

- California (`CA`)
- Oregon (`OR`)
- Washington (`WA`)

Zero records for OR or WA is an accepted source-data outcome.

However:

- The complete batch must not be empty.
- CA is expected to contain records.
- The pipeline should fail or raise an alert if CA unexpectedly contains zero records.

### Freshness SLA

**Target freshness:** Daily by 06:00 UTC.

For this take-home implementation, the pipeline is executed from the Databricks notebook.

## 5. Versioning

The current Gold data product version is **v1**.

Gold path:

`/Volumes/university_chapters/data_product/project_files/gold/university_chapters/v1`

Backward-compatible changes may be introduced within the existing version.

Breaking schema or contract changes should be released as a new major version, such as `v2`.

Consumers should reference the versioned Gold path rather than relying on an unversioned location.

## 6. Data Classification

**Classification:** Public data

The source is a public ArcGIS dataset.

No PII is expected in the published Gold data product.

## 7. Quarantine Interface

Quarantined records are stored separately from the Gold data product.

**Quarantine Path:**

`/Volumes/university_chapters/data_product/project_files/quarantine/university_chapters/<run_id>`

Quarantine records contain sufficient information to investigate the DQ failure, including:

- `ingest_run_id`
- `chapter_id`
- source attributes
- DQ reason code
- raw payload

Quarantined records must never be published to Gold.

## 8. Audit Metrics

The pipeline records the following metrics for each ingestion run:

- `rows_in`
- `rows_quarantined`
- `rows_warned`
- `rows_ok`

These metrics provide visibility into data quality and processing outcomes for each run.
