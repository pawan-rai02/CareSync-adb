# CareSync-adb

An end-to-end **healthcare data lakehouse** built on **Azure Databricks** for the fictional *CareSync Health Network*. Raw operational data about hospitals, doctors, patients, insurance providers and lab results is ingested from multiple source systems, cleaned, modelled and turned into analytics-ready **gold** tables using the **Medallion Architecture** (Landing -> Bronze -> Silver -> Gold) on **Delta Lake** and **Unity Catalog**.

This README is written so that anyone (a new team member, a reviewer, or a learner) can understand **what** the project does, **why** each step exists and **how** to run or extend it.

---

## Table of contents

1. [Aim of the project](#1-aim-of-the-project)
2. [Objectives](#2-objectives)
3. [Business problem and use cases](#3-business-problem-and-use-cases)
4. [High-level architecture](#4-high-level-architecture)
5. [Key concepts explained](#5-key-concepts-explained)
6. [Repository structure](#6-repository-structure)
7. [Data sources](#7-data-sources)
8. [Unity Catalog layout](#8-unity-catalog-layout)
9. [Step-by-step pipeline walkthrough](#9-step-by-step-pipeline-walkthrough)
10. [Data model reference](#10-data-model-reference)
11. [How to run the project](#11-how-to-run-the-project)
12. [Data quality and validation](#12-data-quality-and-validation)
13. [Design decisions](#13-design-decisions)
14. [Troubleshooting](#14-troubleshooting)
15. [How to extend the project](#15-how-to-extend-the-project)
16. [Glossary](#16-glossary)

---

## 1. Aim of the project

**To build a reliable, reusable and well-governed data platform that turns scattered raw healthcare data into trusted, analytics-ready datasets, so that hospital operations, clinical quality and patient insights can be measured from a single source of truth.**

---

## 2. Objectives

| # | Objective | How the project achieves it |
| --- | --- | --- |
| 1 | **Ingest data from multiple source systems** | Data from a PostgreSQL database (Neon) and CSV files in ADLS is copied into a Unity Catalog landing volume and loaded into bronze. |
| 2 | **Keep a complete, auditable history of raw data** | Bronze is append-only; every batch is tagged with `landing_timestamp` and `insert_timestamp`. |
| 3 | **Produce clean, de-duplicated, current-state data** | Silver uses configurable FULL / APPEND / MERGE (upsert) loads keyed on business keys. |
| 4 | **Use generic, parameter-driven notebooks** | One notebook per layer works for every table; behaviour is controlled by widgets / job parameters. |
| 5 | **Provide a shared date dimension** | A calendar table (1940 to 2030) supports date-based reporting and joins. |
| 6 | **Deliver business-ready gold tables** | Three gold tables (patient 360, hospital daily summary, lab test trends) are built from a documented source-to-target mapping (STTM). |
| 7 | **Be resilient and observable** | Notebooks skip empty batches cleanly, validate inputs, print readable step logs and return record counts to the orchestrator. |
| 8 | **Govern everything centrally** | All data lives in Unity Catalog (`adb_db_caresync`) with separate schemas per layer. |

---

## 3. Business problem and use cases

Healthcare data is usually spread across systems: hospital and patient master data in an operational database, lab results and insurance data delivered as files. Without a unified platform it is hard to answer basic questions consistently.

This project enables questions such as:

* **Patient view:** Who is this patient, which hospital and insurer do they belong to, how many lab tests have they had and how many were abnormal or critical? (`gold.patient_360`)
* **Hospital operations:** How many tests did each hospital run per day, how many doctors were active, and how did volume change compared with the previous day? (`gold.hospital_daily_summary`)
* **Clinical quality trends:** For each test type, how is the abnormal rate trending month over month? (`gold.lab_test_trends`)

---

## 4. High-level architecture

```
  SOURCE SYSTEMS              LANDING                  BRONZE                  SILVER                    GOLD
 +-----------------+    +--------------------+   +------------------+   +--------------------+   +-------------------------+
 | Neon PostgreSQL |    | UC Volume          |   | Raw Delta tables |   | Clean Delta tables |   | Business tables         |
 |  - hospitals    |--->| landing_volume/    |-->| append-only      |-->| FULL/APPEND/MERGE  |-->|  - patient_360          |
 |  - doctors      |    |  <source>/<table>/ |   | + landing_ts     |   | + insert/update ts |   |  - hospital_daily_summary|
 |  - patients     |    |  <timestamp>/      |   | + insert_ts      |   | + calendar dim     |   |  - lab_test_trends      |
 +-----------------+    |  (parquet / csv)   |   +------------------+   +--------------------+   +-------------------------+
 | ADLS CSV files  |--->|                    |
 |  - insurance_   |    +--------------------+
 |    providers    |
 |  - lab_results  |         Orchestration: an external pipeline (Azure Data Factory) copies files
 +-----------------+         to landing and triggers the Databricks notebooks with parameters.
```

**Flow in one sentence:** files land in a volume, `Landing_to_bronze` appends them to bronze, `Bronze to silver` cleans and upserts them into silver, and the `Gold/NB_*` notebooks rebuild aggregated business tables from silver.

---

## 5. Key concepts explained

If you are new to lakehouses, read this section first.

* **Lakehouse:** A platform that combines cheap file storage (data lake) with database features such as transactions, schemas and SQL (data warehouse).
* **Delta Lake:** The table format used for every table here. It adds ACID transactions, versioning (time travel), `MERGE` support and table history on top of Parquet files.
* **Unity Catalog (UC):** Databricks' governance layer. Data is organised as `catalog.schema.table`, e.g. `adb_db_caresync.silver.patients`. Files are stored in **volumes**, e.g. `/Volumes/adb_db_caresync/landing/landing_volume`.
* **Medallion Architecture:** A layered design where data quality improves at each step:
  * **Landing:** exact files as delivered by the source. Nothing changed.
  * **Bronze:** the same raw data stored as Delta tables, plus metadata on when it arrived. Keeps full history.
  * **Silver:** cleaned, de-duplicated, current-state data per entity. Query-ready.
  * **Gold:** aggregated, joined, business-friendly tables designed for reporting and dashboards.
* **Widgets / parameters:** Inputs to a notebook (`dbutils.widgets`). They let the same notebook process any table, and let a job or orchestrator pass values in.
* **Upsert (MERGE):** Update a row if its key already exists, otherwise insert it.
* **STTM (Source-To-Target Mapping):** A specification document that lists, for each target column, its source column(s) and transformation rule.

---

## 6. Repository structure

```
CareSync-adb/
|-- README.md                       <- this document
|-- Landing_to_bronze               <- notebook: landing files -> bronze tables (generic)
|-- Bronze to silver                <- notebook: bronze -> silver with FULL/APPEND/MERGE (generic)
|-- Prepare calendar table          <- notebook: builds silver.calendar date dimension
|-- test                            <- scratch notebook for ad-hoc checks (not part of the pipeline)
`-- Gold/
    |-- sttm_silver_to_gold.xlsx    <- source-to-target mapping for the gold layer
    |-- NB_patient_360              <- notebook: builds gold.patient_360
    |-- NB_hospital_daily_summary   <- notebook: builds gold.hospital_daily_summary
    `-- NB_lab_test_trends          <- notebook: builds gold.lab_test_trends
```

---

## 7. Data sources

| Source system | Folder in landing volume | Tables | File format |
| --- | --- | --- | --- |
| Neon PostgreSQL (operational DB) | `neon_postgres/` | `hospitals`, `doctors`, `patients` | Parquet |
| Azure Data Lake Storage files | `adls_csv/` | `insurance_providers`, `lab_results` | CSV (with header) |

**Landing path convention**

```
/Volumes/adb_db_caresync/landing/landing_volume/<source_system>/<table_name>/<landing_timestamp>/<files>

example:
/Volumes/adb_db_caresync/landing/landing_volume/neon_postgres/hospitals/2026-11-06_06:46:14/data_....parquet
```

Each load writes into a **new timestamped folder**. This means every batch is isolated, can be re-processed independently, and is easy to trace back to the run that produced it.

---

## 8. Unity Catalog layout

| Object | Purpose |
| --- | --- |
| Catalog `adb_db_caresync` | Container for the whole project |
| Schema `landing` + volume `landing_volume` | Raw files exactly as received |
| Schema `bronze` | Raw Delta tables, append-only (5 tables) |
| Schema `silver` | Clean, current-state Delta tables + `calendar` (6 tables) |
| Schema `gold` | Business / reporting tables (3 tables) |

---

## 9. Step-by-step pipeline walkthrough

### Step 0: Files arrive in the landing volume

**What:** An external orchestrator (Azure Data Factory) copies data from PostgreSQL and ADLS into the landing volume under a new `<landing_timestamp>` folder, then calls the Databricks notebooks with parameters.

**Why:** Keeping an untouched copy of every delivery means you can always re-run downstream steps or investigate data issues against the original files.

---

### Step 1: Landing -> Bronze (`Landing_to_bronze`)

**Goal:** Load one batch of landed files into the matching bronze Delta table without changing the data.

**Parameters**

| Parameter | Meaning | Example |
| --- | --- | --- |
| `source_system` | Folder of the source system in the landing volume | `neon_postgres` |
| `table_name` | Table being loaded (same name in landing and bronze) | `hospitals` |
| `landing_timestamp` | Batch folder to read | `2026-11-06_06:46:14` |
| `source_file_format` | `csv` or anything else (treated as `parquet`) | `parquet` |

**What happens, cell by cell**

1. **Read parameters** from widgets.
2. **Build the landing path** `/Volumes/adb_db_caresync/landing/landing_volume/{source_system}/{table_name}/{landing_timestamp}/`.
3. **Guard against empty batches:** if the folder does not exist or is empty (e.g. the orchestrator found no new incremental data), the notebook exits cleanly with `SKIPPED: no landing data at ...` instead of failing. This keeps the overall pipeline green when there is simply nothing to load.
4. **Read the files:** CSV is read with `header=true` and `inferSchema=true`; otherwise Parquet is read (Parquet already carries its schema).
5. **Add metadata columns:**
   * `landing_timestamp`: which batch the row came from (used by the next step to pick exactly this batch).
   * `insert_timestamp`: when the row was written to bronze.
6. **Append** to `adb_db_caresync.bronze.<table_name>`. Bronze is never overwritten, so it holds the full history of all deliveries.

**Why append-only?** Bronze is the system of record for raw data. If silver logic ever has a bug, silver can be rebuilt from bronze without going back to the source.

---

### Step 2: Bronze -> Silver (`Bronze to silver`)

**Goal:** Take one batch from bronze and write it to silver using the right load strategy, producing clean, current-state tables.

**Parameters**

| Parameter | Meaning | Example |
| --- | --- | --- |
| `table_name` | Table name (same in bronze and silver) | `patients` |
| `landing_timestamp` | Batch to pick from bronze | `2026-11-06_06:46:14` |
| `load_type` | `FULL`, `APPEND` or `MERGE` (case and spaces ignored) | `MERGE` |
| `merge_keys` | Comma-separated key column(s), required only for `MERGE` | `patient_id` |

**What happens, step by step**

1. **Read parameters** and print them, warning if any are empty.
2. **Read only this batch** from bronze: `WHERE landing_timestamp = '<parameter>'`. Older batches are not re-processed.
3. **Add audit columns:**
   * `insert_timestamp`: when the row was first written to silver.
   * `update_timestamp`: when the row was last changed in silver.
4. **Write to silver** depending on `load_type`:

| Load type | What happens to the silver table | Typical use |
| --- | --- | --- |
| `FULL` | Table is overwritten with only this batch | Small reference tables delivered in full each time |
| `APPEND` | Rows are added on top of existing data (table created if missing) | Event / log data where rows never change |
| `MERGE` | Upsert on `merge_keys`: existing keys updated, new keys inserted | Master data that changes over time (patients, doctors, hospitals) |

   **MERGE details**
   * Validates that `merge_keys` is provided and that every key column exists; otherwise stops with a clear error.
   * **First run:** if the silver table does not exist yet, it is simply created from the batch.
   * **Later runs:**
     * Duplicate keys within the batch are removed (`dropDuplicates(keys)`), because a Delta MERGE fails if one target row matches multiple source rows.
     * Matched rows: all columns are updated **except** the key columns and `insert_timestamp` (so the original creation time is preserved).
     * Unmatched rows: inserted.
     * The number of rows updated and inserted is read from the Delta table history and printed.
   * Any other `load_type` value raises `Invalid load_type ... Expected one of: FULL, APPEND, MERGE`.
5. **Optional quick check:** a sample query against a silver table (hard-coded to `hospitals`; change or skip in production).
6. **Return the record count** with `dbutils.notebook.exit(<count>)` so the orchestrator can log or check it.

---

### Step 3: Calendar dimension (`Prepare calendar table`)

**Goal:** Build `adb_db_caresync.silver.calendar`, one row per day from **1940-01-01 to 2030-12-31** (33,238 rows).

**Why:** Gold tables need consistent date attributes (day name, weekend flag, month name, quarter). Computing them once in a shared dimension avoids repeating logic and guarantees consistency. The early start date covers patient dates of birth.

**Key columns**

* `date_key` (int, `yyyyMMdd`, e.g. `20240101`): the join key used by gold notebooks.
* Year: `year`, `is_leap_year`, `first_day_of_year`, `last_day_of_year`.
* Quarter: `quarter`, `quarter_name` (`Q1`), `year_quarter` (`2024-Q1`), `day_of_quarter`, quarter start/end dates.
* Month: `month`, `month_name`, `month_short_name`, `year_month`, `year_month_key`, `days_in_month`, month start/end dates.
* Week (ISO 8601, Monday start): `week_of_year`, `iso_year`, `week_of_month`, `week_start_date`, `week_end_date`.
* Day: `day_of_month`, `day_of_year`, `day_of_week` (1 = Monday ... 7 = Sunday), `day_name`, `day_short_name`.
* Flags: `is_weekend`, `is_weekday`, `is_month_start/end`, `is_quarter_start/end`, `is_year_start/end`.

**Load:** full overwrite (`overwriteSchema=true`). It only needs to run once, or again if the date range or columns change.

**Built-in validation:** row count equals distinct keys equals the expected number of days, and the number of leap days (23) is correct.

---

### Step 4: Silver -> Gold (`Gold/NB_*`)

**Goal:** Build business-ready tables defined in `Gold/sttm_silver_to_gold.xlsx`.

**Common pattern for all gold notebooks**

* Full rebuild with `CREATE OR REPLACE TABLE ... AS SELECT` (simple, idempotent, always consistent with silver).
* Start from a **driving table** that defines the grain, then `LEFT JOIN` lookups so no rows are lost when a lookup is missing.
* Finish by returning the gold row count with `dbutils.notebook.exit`.

#### 4a. `NB_patient_360` -> `gold.patient_360`

* **Grain:** one row per patient.
* **Driving table:** `silver.patients` (all patients kept).
* **Joins (LEFT):**
  * `silver.hospitals` on `registered_hospital_id = hospital_id` -> `hospital_name`
  * `silver.insurance_providers` on `insurance_provider_id` -> `insurance_provider_name`
  * `silver.lab_results` pre-aggregated per `patient_id`
* **Derived columns:**
  * `age` = floor(days between `date_of_birth` and today / 365.25)
  * `total_tests`, `abnormal_test_count` (`result_status = 'Abnormal'`), `critical_test_count` (`result_status = 'Critical'`), defaulting to 0 for patients without tests
  * `last_test_date` = latest `test_date`
* **Note:** `date_of_birth` is a timestamp in silver and is cast to `date`.
* **Validation:** gold row count = `silver.patients` count, `patient_id` is unique, and `SUM(total_tests)` = `silver.lab_results` count.

#### 4b. `NB_hospital_daily_summary` -> `gold.hospital_daily_summary`

* **Grain:** one row per hospital per day that has lab tests.
* **Driving table:** `silver.lab_results` grouped by `hospital_id`, `test_date`.
* **Joins (LEFT):** `silver.hospitals` for `hospital_name`; `silver.calendar` on `date_key = CAST(date_format(test_date,'yyyyMMdd') AS INT)` for `day_name`, `is_weekend`.
* **Metrics:** `total_tests`, `unique_patients_tested`, `active_doctor_count`, `abnormal_test_count`, `critical_test_count`, `abnormal_rate_pct`, `critical_rate_pct`.
* **Trend:** `prev_day_total_tests` via `LAG(total_tests)` per hospital ordered by date (previous day *with tests*), and `test_volume_change_pct` = (today - previous) / previous x 100 (NULL when there is no previous day or it is 0).

#### 4c. `NB_lab_test_trends` -> `gold.lab_test_trends`

* **Grain:** one row per test type (`test_name`) per calendar month.
* **Driving table:** `silver.lab_results` grouped by `test_name`, year and month of `test_date`.
* **Join (LEFT):** `silver.calendar` on the first day of the month -> `month_name`, `quarter_name`.
* **Metrics:** `total_tests`, `abnormal_test_count`, `critical_test_count`, `abnormal_rate_pct`, `critical_rate_pct`.
* **Trend:** `prev_month_abnormal_rate_pct` via `LAG` per test, and `abnormal_rate_change_pct` = current rate - previous month rate (percentage points).

---

## 10. Data model reference

### Silver entities and relationships

```
 insurance_providers (insurance_provider_id)
          ^
          |
 patients (patient_id) ---registered_hospital_id---> hospitals (hospital_id) <--- doctors (doctor_id, hospital_id)
          ^                                                ^                          ^
          |                                                |                          |
          +--------------- lab_results (lab_result_id) ----+--------------------------+
                           patient_id, hospital_id, doctor_id, test_date

 calendar (date_key = yyyyMMdd) <--- joined on dates in gold
```

| Silver table | Business key | Main columns | Approx. rows |
| --- | --- | --- | --- |
| `hospitals` | `hospital_id` | `hospital_name`, `city`, `state`, `hospital_type`, `bed_count` | 51 |
| `doctors` | `doctor_id` | `first_name`, `last_name`, `specialty`, `hospital_id`, `phone_number` | 502 |
| `patients` | `patient_id` | `first_name`, `last_name`, `date_of_birth`, `gender`, `city`, `state`, `registered_hospital_id`, `insurance_provider_id` | 5,003 |
| `insurance_providers` | `insurance_provider_id` | `provider_name`, `provider_type`, `coverage_region`, `contact_email` | 25 |
| `lab_results` | `lab_result_id` | `patient_id`, `doctor_id`, `hospital_id`, `test_name`, `test_date`, `result_value`, `result_unit`, `result_status`, `reported_at` | 8,248 |
| `calendar` | `date_key` | see Step 3 | 33,238 |

All silver tables (except `calendar`) also carry `landing_timestamp`, `insert_timestamp` and `update_timestamp`. Source tables from PostgreSQL additionally carry `created_at` and `updated_at`.

### Gold tables

| Gold table | Grain | Built by |
| --- | --- | --- |
| `patient_360` | 1 row per patient | `Gold/NB_patient_360` |
| `hospital_daily_summary` | 1 row per hospital per test day | `Gold/NB_hospital_daily_summary` |
| `lab_test_trends` | 1 row per test type per month | `Gold/NB_lab_test_trends` |

Row counts are as of the latest run and will change as new data arrives.

---

## 11. How to run the project

### Prerequisites

* An Azure Databricks workspace with Unity Catalog enabled.
* Catalog `adb_db_caresync` with schemas `landing`, `bronze`, `silver`, `gold` and the volume `landing.landing_volume`.
* Permissions to read the volume and create tables in the bronze, silver and gold schemas.
* Serverless compute or a cluster running a recent Databricks Runtime.

### Recommended run order

| Order | Notebook | Run frequency | Example parameters |
| --- | --- | --- | --- |
| 1 | `Prepare calendar table` | Once (or when the date range changes) | none |
| 2 | `Landing_to_bronze` | Per table, per batch | `source_system=neon_postgres`, `table_name=patients`, `landing_timestamp=<batch>`, `source_file_format=parquet` |
| 3 | `Bronze to silver` | Per table, per batch, after step 2 | `table_name=patients`, `landing_timestamp=<batch>`, `load_type=MERGE`, `merge_keys=patient_id` |
| 4 | `Gold/NB_patient_360`, `Gold/NB_hospital_daily_summary`, `Gold/NB_lab_test_trends` | After all silver loads finish (can run in parallel) | none |

### Suggested silver configuration per table

| Table | Source system | Format | load_type | merge_keys |
| --- | --- | --- | --- | --- |
| `hospitals` | `neon_postgres` | parquet | MERGE | `hospital_id` |
| `doctors` | `neon_postgres` | parquet | MERGE | `doctor_id` |
| `patients` | `neon_postgres` | parquet | MERGE | `patient_id` |
| `insurance_providers` | `adls_csv` | csv | MERGE | `insurance_provider_id` |
| `lab_results` | `adls_csv` | csv | MERGE | `lab_result_id` |

### Running manually (interactive)

1. Open the notebook in Databricks.
2. Fill in the widgets at the top of the notebook.
3. Click **Run all** and read the printed step summaries.

### Running from an orchestrator

Pass the same values as notebook / job task parameters (Azure Data Factory Notebook activity, or a Lakeflow Job). Each notebook returns a value via `dbutils.notebook.exit`:

* `Landing_to_bronze`: `SKIPPED: ...` when no data was landed.
* `Bronze to silver` and gold notebooks: the processed record count.

---

## 12. Data quality and validation

| Check | Where | What it protects against |
| --- | --- | --- |
| Missing / empty landing folder -> clean skip | `Landing_to_bronze` | Pipeline failures on days with no new data |
| Empty parameter warnings | `Bronze to silver` | Misconfigured job runs |
| `merge_keys` present and exist in data | `Bronze to silver` | Silent bad merges |
| De-duplicate source keys before MERGE | `Bronze to silver` | Delta multiple-match MERGE errors |
| Preserve `insert_timestamp` on update | `Bronze to silver` | Losing original creation time |
| Row count = distinct keys = expected days, leap-day count | `Prepare calendar table` | Gaps or duplicates in the date dimension |
| Gold rows = silver patients, unique `patient_id`, total tests reconcile | `NB_patient_360` | Join fan-out or lost rows |
| LEFT JOINs from driving table | All gold notebooks | Dropping facts when a lookup is missing |
| Divide-by-zero guard in change % | `NB_hospital_daily_summary` | Errors / infinite values |

---

## 13. Design decisions

* **Generic notebooks over one notebook per table:** less code, consistent behaviour, and new tables can be added with configuration only.
* **`landing_timestamp` as the batch identifier:** links a row in silver back to its bronze batch and the original landing folder (full lineage).
* **Append-only bronze:** guarantees raw history and makes silver rebuildable.
* **MERGE for master data:** silver always shows the latest state per key without duplicates.
* **Full rebuild for gold:** the data volume is small, so `CREATE OR REPLACE` is simpler and safer than incremental logic and always reflects the current silver state.
* **Integer `date_key` (`yyyyMMdd`):** compact, human-readable and fast to join.
* **STTM-driven gold:** business rules are documented outside the code, making reviews and changes easier.

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `SKIPPED: no landing data at ...` | No files for that batch | Expected when there is no new data; confirm the copy step if data was expected |
| `Invalid load_type ''` | `load_type` widget / parameter is empty | Pass `FULL`, `APPEND` or `MERGE` |
| `merge_keys must be provided ...` | MERGE without keys | Pass the business key, e.g. `patient_id` |
| `Merge keys not found in source data` | Typo or wrong table | Check column names in the bronze table |
| Bronze read returns 0 rows | `landing_timestamp` does not match exactly | Use the exact folder name used in landing |
| CSV columns have wrong types | `inferSchema` guessed incorrectly | Provide an explicit schema or cast in silver |
| Gold row counts unexpectedly high | Join fan-out | Check that lookup tables are unique on their join key |

---

## 15. How to extend the project

**Add a new source table**

1. Land files under `landing_volume/<source_system>/<new_table>/<timestamp>/`.
2. Run `Landing_to_bronze` with the new `table_name` (bronze table is created automatically on first append).
3. Run `Bronze to silver` with an appropriate `load_type` and `merge_keys`.
4. Add the table to your orchestrator configuration.

**Add a new gold table**

1. Add the target definition (grain, driving table, joins, column rules) to `Gold/sttm_silver_to_gold.xlsx`.
2. Create `Gold/NB_<table_name>` following the existing pattern: overview markdown, CTAS build, validation query, return record count.
3. Schedule it after the silver loads.

**Ideas for future improvements**

* Explicit schemas and data quality expectations for CSV sources.
* SCD Type 2 history in silver for slowly changing attributes (e.g. patient address).
* Incremental gold loads if data volume grows.
* Dashboards on top of the gold tables.
* CI/CD with Declarative Automation Bundles.

---

## 16. Glossary

| Term | Meaning |
| --- | --- |
| ADF | Azure Data Factory, the orchestration / copy service |
| ADLS | Azure Data Lake Storage |
| CTAS | `CREATE TABLE AS SELECT` |
| Delta Lake | Open table format with ACID transactions, used for all tables |
| Driving table | The table that defines the grain of a gold table |
| Grain | What one row represents (e.g. one patient, one hospital per day) |
| Medallion | Landing / Bronze / Silver / Gold layering |
| MERGE / Upsert | Update existing rows, insert new ones |
| STTM | Source-To-Target Mapping specification |
| UC | Unity Catalog, Databricks governance layer |
| Volume | Unity Catalog object for storing files |
