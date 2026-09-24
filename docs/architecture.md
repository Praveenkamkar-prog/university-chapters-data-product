# University Chapters Data Product — Architecture

## 1. Architecture Overview

The pipeline follows a simple Medallion architecture using Databricks and Spark transformations.

Architecture flow:
```text
ArcGIS REST API
       |
       v
   BRONZE
Raw API payload + ingest metadata
       |
       v
   SILVER
Parse JSON
Clean and type columns
Deduplicate chapters
       |
       v
  DATA QUALITY
       |
       +----------------------+
       |                      |
       v                      v
 QUARANTINE              VALID RECORDS
Invalid coordinates     OK / WARNING
                              |
                              v
                            GOLD
                     Consumer-ready data product

```

## 2. Source

The source is the public ArcGIS UniversityChapters_Public REST API.

The pipeline requests records for only:

- California (CA)
- Oregon (OR)
- Washington (WA)

The source response contains chapter attributes and geometry coordinates.


## 3. Bronze Layer

Bronze stores the API response in a raw or near-as-received form.

Each record contains:

- ingest_run_id
- ingest_timestamp
- source_name
- source_url
- raw_payload

Bronze data is stored by ingestion run so that historical source payloads can
be retained for troubleshooting and traceability.

Example path:

bronze/university_chapters/<run_id>/

No business transformations are applied in Bronze.


## 4. Silver Layer

Silver converts the raw API payload into a structured dataset.

Transformations include:

- JSON parsing
- Column mapping
- Data type conversion
- Trimming text values
- Standardizing state values to uppercase
- Extracting longitude and latitude
- Deduplication at the chapter_id grain

The main Silver columns include:

- chapter_id
- chapter_name
- city
- state
- longitude
- latitude
- source_object_id
- ingest_run_id
- ingest_timestamp
- raw_payload


## 5. Data Quality Processing

Two data-quality rules are implemented.

### DQ-Q1 — Invalid Coordinates

Records with missing or invalid coordinates are treated as hard failures.

They are removed from the valid Silver dataset and written to quarantine with:

dq_reason_code = INVALID_COORDINATES

### DQ-W1 — Missing or Unknown City

Records where city is null, blank, or UNKNOWN are treated as warnings.

They remain eligible for Gold publication with:

dq_status = WARNING

dq_warnings = ["MISSING_OR_UNKNOWN_CITY"]

Clean records receive:

dq_status = OK

dq_warnings = []


## 6. Quarantine Layer

Quarantined records are stored separately by ingestion run.

Example path:

quarantine/university_chapters/<run_id>/

The quarantine dataset retains sufficient information to investigate the failed
record, including the raw payload and DQ reason.

Quarantined records are never published to Gold.


## 7. Gold Layer

Gold is the consumer-facing data product.

The Gold dataset contains only valid records that passed the hard DQ rule.

Warning records are allowed into Gold and are clearly identified using
dq_status and dq_warnings.

Gold is versioned:

gold/university_chapters/v1/

The Gold grain is one row per chapter_id.


## 8. Idempotency and Reruns

Bronze retains data by ingest_run_id.

Silver and Gold use overwrite behavior for their current output paths,
making repeated execution deterministic for the same processing scope.

This avoids accumulating duplicate Gold records across notebook reruns.

A production implementation could use Delta Lake and MERGE based on chapter_id
for stronger incremental and idempotent processing.


## 9. Technology

The implementation uses:

- Databricks
- Apache Spark / PySpark
- Unity Catalog Volume
- Parquet
- Python
- ArcGIS REST API

The take-home implementation uses a Unity Catalog Volume to keep the setup
simple and reproducible.


## 10. Production Considerations

For a production deployment, the architecture could be extended with:

- ADLS Gen2 as the storage layer
- Delta Lake tables
- ADF or another orchestrator
- CI/CD
- Secret management
- Centralized monitoring and alerting
- Production data catalog and governance
