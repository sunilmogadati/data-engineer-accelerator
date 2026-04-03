# Star Schema Design — The End-to-End Pipeline

**From source systems to dashboards — every step, every tool, every decision.**

---

## The Complete Flow

```mermaid
graph TD
    subgraph "SOURCE SYSTEMS"
        S1["VA Platform<br/>(JSON via API)"]
        S2["Call Management<br/>(SQL Server)"]
        S3["Order System<br/>(SQL Server)"]
        S4["Payment Gateway<br/>(API)"]
        S5["Phone Config<br/>(SQL Server)"]
    end

    subgraph "STEP 1: INGEST → Bronze (GCS)"
        S1 -->|"Cloud Function<br/>triggered by webhook"| B1["gs://bucket/bronze/<br/>va_calls/"]
        S2 -->|"Scheduled extract<br/>(Dataflow or<br/>Cloud Function)"| B2["gs://bucket/bronze/<br/>calls/"]
        S3 -->|"Scheduled extract"| B3["gs://bucket/bronze/<br/>orders/"]
        S4 -->|"API pull"| B4["gs://bucket/bronze/<br/>payments/"]
        S5 -->|"Scheduled extract"| B5["gs://bucket/bronze/<br/>dnis_sources/"]
    end

    subgraph "STEP 2: CLEAN → Silver (BigQuery)"
        B1 --> SV["silver.calls<br/>Dedup, timezone,<br/>type fix, null flag,<br/>VA + Live unified"]
        B2 --> SV
        B3 --> SV2["silver.orders<br/>Dedup, type fix"]
        B4 --> SV3["silver.payments<br/>Dedup, status normalize"]
        B5 --> SV4["silver.dnis_sources<br/>Validated config"]
    end

    subgraph "STEP 3: MODEL → Gold Dimensions (BigQuery)"
        SV4 --> DIM1["dim_date"]
        SV4 --> DIM2["dim_time"]
        SV4 --> DIM3["dim_campaign"]
        SV --> DIM4["dim_disposition"]
        DIM5["dim_agent"]
    end

    subgraph "STEP 4: LOAD → Gold Facts (BigQuery)"
        SV --> FACT["fact_calls"]
        SV2 --> FACT
        DIM1 --> FACT
        DIM2 --> FACT
        DIM3 --> FACT
        DIM4 --> FACT
        DIM5 --> FACT
    end

    subgraph "STEP 5: SERVE → Dashboards & APIs"
        FACT --> DASH["Looker Studio<br/>Dashboard"]
        FACT --> API["REST API<br/>(FastAPI / Cloud Run)"]
        FACT --> ML["ML Feature Store<br/>(for AI pipeline)"]
    end

    subgraph "ORCHESTRATION (Cloud Composer / Airflow)"
        ORCH["DAG: runs nightly<br/>Step 1 → 2 → 3 → 4 → 5<br/>with data quality gates"]
    end
```

---

## Step 1: INGEST — Source Systems → Bronze (GCS)

**Goal:** Get raw data from every source system into cloud storage. No transformation. Exact copy.

**Where it lands:** GCS (Google Cloud Storage) bucket, organized by source and date.

```
gs://call-center-pipeline/
  bronze/
    va_calls/
      2026-04-03/va_calls_20260403.json       ← Daily extract from VA platform API
    calls/
      2026-04-03/calls_20260403.csv           ← Daily extract from SQL Server
    orders/
      2026-04-03/orders_20260403.csv
    payments/
      2026-04-03/payments_20260403.csv
    dnis_sources/
      dnis_sources_latest.csv                  ← Config table — overwritten, not partitioned by date
```

### How Data Gets Into Bronze

| Source | Ingestion Method | GCP Service | Trigger |
|:---|:---|:---|:---|
| **VA Platform** (real-time, JSON via webhook) | Webhook hits a Cloud Function that writes to GCS | **Cloud Functions** | Event: VA platform sends `call_analyzed` webhook |
| **SQL Server tables** (batch, scheduled) | Extract query dumps CSV to GCS | **Dataflow** (managed Apache Beam) or **Cloud Function** with SQL connector | Schedule: Cloud Scheduler triggers at 2 AM daily |
| **API sources** (payments, third-party) | Cloud Function calls API, writes response to GCS | **Cloud Functions** | Schedule: Cloud Scheduler |
| **Config/lookup tables** (DNIS mapping) | Full extract — small table, overwrite daily | **Cloud Function** or manual upload | Schedule or on-change |

### GCP Console UI: Setting Up the Bucket

1. Go to [console.cloud.google.com/storage](https://console.cloud.google.com/storage)
2. **Create Bucket** → name: `call-center-pipeline` → location: `us-central1` → **Create**
3. Inside the bucket, **Create Folder**: `bronze`
4. Inside `bronze`, create folders: `va_calls`, `calls`, `orders`, `payments`, `dnis_sources`

### CLI

```bash
gcloud storage buckets create gs://call-center-pipeline/ --location=us-central1
gcloud storage cp data/calls.json gs://call-center-pipeline/bronze/va_calls/
gcloud storage cp data/campaigns.csv gs://call-center-pipeline/bronze/dnis_sources/
gcloud storage cp data/orders.csv gs://call-center-pipeline/bronze/orders/
gcloud storage cp data/products.csv gs://call-center-pipeline/bronze/products/
```

### Cloud Function Example: Ingest VA Calls from Webhook

```python
# cloud_function_ingest_va.py
# Triggered when the VA platform sends a call_analyzed webhook
# Writes the raw JSON to GCS Bronze layer

import json
from google.cloud import storage
from datetime import datetime

def ingest_va_call(request):
    """Cloud Function triggered by HTTP webhook from VA platform."""
    call_data = request.get_json()
    call_id = call_data.get("call_id", "unknown")
    today = datetime.utcnow().strftime("%Y-%m-%d")

    # Write raw JSON to GCS — no transformation
    client = storage.Client()
    bucket = client.bucket("call-center-pipeline")
    blob = bucket.blob(f"bronze/va_calls/{today}/{call_id}.json")
    blob.upload_from_string(json.dumps(call_data), content_type="application/json")

    return {"status": "ingested", "call_id": call_id}, 200
```

### GCS Event Trigger (Alternative: Trigger Pipeline When File Lands)

Instead of scheduling, the pipeline can trigger when new data arrives:

```bash
# Create a Cloud Function that triggers when a file lands in bronze/
gcloud functions deploy trigger_pipeline \
    --runtime python310 \
    --trigger-resource call-center-pipeline \
    --trigger-event google.storage.object.finalize \
    --entry-point on_file_uploaded
```

This fires every time a new file is uploaded to the bucket. The function checks which folder the file landed in and triggers the appropriate pipeline step.

---

## Step 2: CLEAN — Bronze → Silver (BigQuery)

**Goal:** Load raw data from GCS into BigQuery, clean it, and produce a unified, deduplicated dataset.

### Load Bronze Files into BigQuery Raw Tables

```bash
# Load from GCS into BigQuery raw tables (auto-detect schema)
bq load --autodetect --replace --source_format=NEWLINE_DELIMITED_JSON \
    raw.va_calls gs://call-center-pipeline/bronze/va_calls/2026-04-03/*.json

bq load --autodetect --replace --source_format=CSV \
    raw.calls gs://call-center-pipeline/bronze/calls/2026-04-03/*.csv

bq load --autodetect --replace --source_format=CSV \
    raw.orders gs://call-center-pipeline/bronze/orders/2026-04-03/*.csv

bq load --autodetect --replace --source_format=CSV \
    raw.dnis_sources gs://call-center-pipeline/bronze/dnis_sources/*.csv
```

### Console UI: External Tables (Alternative — Query GCS Directly)

BigQuery can query files in GCS without loading them first — **external tables:**

1. BigQuery → **Create Table** → Source: **Google Cloud Storage**
2. URI: `gs://call-center-pipeline/bronze/va_calls/*.json`
3. File format: **JSON (Newline delimited)**
4. Table type: **External table** (not native — data stays in GCS)
5. This avoids the load step for ad-hoc exploration, but is slower for repeated queries

### Silver Transformation: Clean and Unify

```sql
-- Create Silver dataset
CREATE SCHEMA IF NOT EXISTS silver OPTIONS (location = 'us-central1');

-- Silver calls: dedup, timezone, type fix, union VA + Live, flag nulls
CREATE OR REPLACE TABLE silver.calls AS

WITH va_cleaned AS (
    SELECT
        call_id,
        CAST(dnis AS STRING) AS dnis,
        caller_phone AS caller_ani,
        -- Timezone: convert UTC → local ONCE, here in Silver
        DATETIME(start_time, 'US/Eastern') AS call_started_local,
        DATETIME(end_time, 'US/Eastern') AS call_ended_local,
        DATE(DATETIME(start_time, 'US/Eastern')) AS call_date_local,
        EXTRACT(HOUR FROM DATETIME(start_time, 'US/Eastern')) AS call_hour_local,
        duration AS duration_sec,
        LOWER(TRIM(disposition)) AS disposition,
        channel AS call_type,
        -- Flag nulls (don't drop)
        duration IS NULL AS has_missing_duration,
        disposition IS NULL AS has_missing_disposition,
        -- Dedup: keep first occurrence per call_id
        ROW_NUMBER() OVER (PARTITION BY call_id ORDER BY start_time) AS row_num
    FROM raw.va_calls
),

-- Add similar CTE for live calls if in a separate table:
-- live_cleaned AS ( ... ),

deduplicated AS (
    SELECT * FROM va_cleaned WHERE row_num = 1
    -- UNION ALL
    -- SELECT * FROM live_cleaned WHERE row_num = 1
)

SELECT * EXCEPT(row_num) FROM deduplicated;

-- Silver orders: dedup, type fix
CREATE OR REPLACE TABLE silver.orders AS
SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_date DESC) AS row_num
FROM raw.orders
QUALIFY row_num = 1;

-- Verify Silver
SELECT
    COUNT(*) AS total_calls,
    COUNTIF(has_missing_duration) AS missing_duration,
    COUNTIF(has_missing_disposition) AS missing_disposition,
    COUNT(DISTINCT call_id) AS unique_calls
FROM silver.calls;
```

**What Silver guarantees:**
- Every call_id appears exactly once (deduplicated)
- Timestamps are in local time (timezone converted)
- Data types are correct (strings → dates, etc.)
- Nulls are flagged with boolean columns (not dropped)
- VA + Live calls are in one table with a `call_type` column
- The Silver table IS the input to both Gold (analytics) and ML (feature engineering)

---

## Step 3: MODEL — Silver → Gold Dimensions (BigQuery)

**Goal:** Build the dimension tables that provide context to the fact table.

```sql
CREATE SCHEMA IF NOT EXISTS gold OPTIONS (location = 'us-central1');
```

The dimension CREATE TABLE statements are in [03_Building_It.md](03_Building_It.md) — `dim_date`, `dim_time`, `dim_campaign`, `dim_disposition`. Run them in this step.

**Key point:** Dimensions are mostly static or slowly changing. `dim_date` is built once for the year. `dim_campaign` changes when a new campaign launches. `dim_disposition` changes when a new outcome type is added. These are not rebuilt nightly — they are maintained as reference data.

---

## Step 4: LOAD — Silver + Dimensions → Gold Facts (BigQuery)

**Goal:** Build the fact table by joining Silver data to dimension keys.

```sql
CREATE OR REPLACE TABLE gold.fact_calls
PARTITION BY call_date_local
CLUSTER BY campaign_key
AS
SELECT
    ROW_NUMBER() OVER (ORDER BY s.call_id) AS call_key,

    -- Dimension keys (integer joins)
    CAST(FORMAT_DATE('%Y%m%d', s.call_date_local) AS INT64) AS date_key,
    s.call_hour_local AS time_key,
    dc.campaign_key,
    dd.disposition_key,

    -- Source reference
    s.call_id,
    s.call_type,
    s.call_date_local,

    -- Measures
    s.duration_sec,
    o.total AS order_total,
    o.order_id,
    CASE WHEN o.order_id IS NOT NULL THEN TRUE ELSE FALSE END AS is_order,

    -- Caller info
    s.dnis,
    s.caller_ani,

    -- Quality flags (carried from Silver)
    s.has_missing_duration,
    s.has_missing_disposition

FROM silver.calls s
LEFT JOIN gold.dim_campaign dc ON s.dnis = dc.dnis
LEFT JOIN gold.dim_disposition dd ON s.disposition = LOWER(dd.disposition_name)
LEFT JOIN silver.orders o ON s.call_id = o.call_id;
```

---

## Step 5: VERIFY — Data Quality Gates

**Goal:** Before anyone queries the Gold tables, verify the pipeline produced correct results.

```sql
-- Gate 1: No duplicate call_ids in fact table
SELECT call_id, COUNT(*) AS dupes
FROM gold.fact_calls
GROUP BY call_id HAVING COUNT(*) > 1;
-- Expected: 0 rows

-- Gate 2: No orphan dimension keys (calls without a campaign match)
SELECT COUNT(*) AS orphan_campaigns
FROM gold.fact_calls WHERE campaign_key IS NULL;
-- Expected: 0 (or a known small number — investigate any non-zero result)

-- Gate 3: Row count matches Silver
SELECT 'silver' AS layer, COUNT(*) AS rows FROM silver.calls
UNION ALL
SELECT 'gold', COUNT(*) FROM gold.fact_calls;
-- Expected: same count (or Gold is slightly less if some calls dropped during join)

-- Gate 4: Revenue matches
SELECT
    'silver_orders' AS source, SUM(total) AS revenue FROM silver.orders
UNION ALL
SELECT 'gold_facts', SUM(order_total) FROM gold.fact_calls WHERE is_order;
-- Expected: same total

-- Gate 5: Date range is complete (no missing days)
SELECT dd.full_date, COUNT(f.call_key) AS calls
FROM gold.dim_date dd
LEFT JOIN gold.fact_calls f ON dd.date_key = f.date_key
WHERE dd.full_date BETWEEN '2026-03-15' AND '2026-03-31'
GROUP BY dd.full_date
ORDER BY dd.full_date;
-- Eyeball: any day with 0 calls when there should be some?
```

**If any gate fails:** The pipeline stops. The old Gold tables remain. No bad data reaches dashboards. Investigate, fix, re-run.

---

## Step 6: SERVE — Gold → Dashboards, APIs, ML

### Option A: Looker Studio Dashboard (GCP Native — Free)

**Looker Studio** (formerly Google Data Studio) connects directly to BigQuery.

1. Go to [lookerstudio.google.com](https://lookerstudio.google.com)
2. **Create → Data Source → BigQuery**
3. Select project → `gold` dataset → `fact_calls` table
4. **Add join:** dim_campaign (on campaign_key), dim_date (on date_key)
5. Build charts: bar chart for campaign conversion, line chart for hourly volume, scorecard for total revenue

| Dashboard Panel | Query Behind It |
|:---|:---|
| **Campaign Conversion Rate** | `GROUP BY campaign_name` → bar chart |
| **Hourly Call Volume** | `GROUP BY hour` → line chart |
| **Revenue by Client** | `GROUP BY client_name, SUM(order_total)` → pie chart |
| **Day-of-Week Pattern** | `GROUP BY day_name` → bar chart |
| **Data Quality Scorecard** | `COUNTIF(campaign_key IS NULL)` → red/green indicator |

### Option B: REST API (Cloud Run + FastAPI)

For applications that need to query the star schema programmatically:

```python
# api.py — FastAPI serving Gold data via REST
from fastapi import FastAPI, Query
from google.cloud import bigquery

app = FastAPI()
client = bigquery.Client()

@app.get("/api/campaign-performance")
def campaign_performance(month: str = Query(default="March")):
    query = f"""
    SELECT dc.campaign_name, dc.channel,
           COUNT(*) AS calls, COUNTIF(f.is_order) AS orders,
           ROUND(COUNTIF(f.is_order)/COUNT(*)*100, 1) AS conversion_pct,
           ROUND(SUM(f.order_total), 2) AS revenue
    FROM gold.fact_calls f
    JOIN gold.dim_campaign dc ON f.campaign_key = dc.campaign_key
    JOIN gold.dim_date dd ON f.date_key = dd.date_key
    WHERE dd.month_name = '{month}'
    GROUP BY dc.campaign_name, dc.channel
    ORDER BY revenue DESC
    """
    results = client.query(query).to_dataframe()
    return results.to_dict(orient="records")
```

Deploy to Cloud Run:
```bash
gcloud run deploy call-center-api --source . --region us-central1 --allow-unauthenticated
```

### Option C: ML Feature Store

The ML pipeline reads from the same Gold tables. Feature extraction is one query:

```sql
-- Extract features for the ML model
SELECT
    f.duration_sec,
    dt.hour AS hour_of_day,
    dd.is_weekend,
    CASE WHEN dc.channel = 'VA' THEN 1 ELSE 0 END AS is_va,
    CASE WHEN f.is_order THEN 1 ELSE 0 END AS placed_order,
    f.order_total,
    f.is_order AS target_conversion  -- The label for the ML model
FROM gold.fact_calls f
JOIN gold.dim_campaign dc ON f.campaign_key = dc.campaign_key
JOIN gold.dim_date dd ON f.date_key = dd.date_key
JOIN gold.dim_time dt ON f.time_key = dt.time_key;
```

This is the bridge between the DE pipeline and the AI pipeline — same Gold tables, different consumer.

---

## Orchestration: Cloud Composer (Airflow)

All steps above run manually today. In production, they run automatically — scheduled nightly, with data quality gates between steps.

**Cloud Composer** is Google's managed Airflow service. An Airflow **DAG (Directed Acyclic Graph, pronounced "dag")** defines the pipeline:

```python
# dags/call_center_pipeline.py
from airflow import DAG
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import GCSToBigQueryOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-engineering',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'call_center_pipeline',
    default_args=default_args,
    schedule_interval='0 3 * * *',    # Run at 3 AM daily
    start_date=datetime(2026, 3, 1),
    catchup=False,
) as dag:

    # Step 1: Load Bronze → BigQuery raw tables
    load_va_calls = GCSToBigQueryOperator(
        task_id='load_va_calls',
        bucket='call-center-pipeline',
        source_objects=['bronze/va_calls/{{ ds }}/*.json'],
        destination_project_dataset_table='raw.va_calls',
        source_format='NEWLINE_DELIMITED_JSON',
        autodetect=True,
        write_disposition='WRITE_TRUNCATE',
    )

    load_orders = GCSToBigQueryOperator(
        task_id='load_orders',
        bucket='call-center-pipeline',
        source_objects=['bronze/orders/{{ ds }}/*.csv'],
        destination_project_dataset_table='raw.orders',
        source_format='CSV',
        autodetect=True,
        write_disposition='WRITE_TRUNCATE',
    )

    # Step 2: Bronze → Silver (dedup, timezone, clean)
    build_silver = BigQueryInsertJobOperator(
        task_id='build_silver',
        configuration={
            'query': {
                'query': open('sql/silver_calls.sql').read(),
                'useLegacySql': False,
            }
        },
    )

    # Step 3: Silver → Gold dimensions (refresh if needed)
    build_dimensions = BigQueryInsertJobOperator(
        task_id='build_dimensions',
        configuration={
            'query': {
                'query': open('sql/gold_dimensions.sql').read(),
                'useLegacySql': False,
            }
        },
    )

    # Step 4: Silver + Dimensions → Gold facts
    build_facts = BigQueryInsertJobOperator(
        task_id='build_facts',
        configuration={
            'query': {
                'query': open('sql/gold_fact_calls.sql').read(),
                'useLegacySql': False,
            }
        },
    )

    # Step 5: Data quality gates
    def verify_data_quality(**context):
        from google.cloud import bigquery
        client = bigquery.Client()

        # Check for duplicates
        dupes = client.query("""
            SELECT COUNT(*) as dupes FROM (
                SELECT call_id, COUNT(*) c
                FROM gold.fact_calls GROUP BY call_id HAVING c > 1
            )
        """).to_dataframe().iloc[0]['dupes']

        if dupes > 0:
            raise ValueError(f"DATA QUALITY FAILURE: {dupes} duplicate call_ids in gold.fact_calls")

        # Check row count matches
        silver_count = client.query("SELECT COUNT(*) c FROM silver.calls").to_dataframe().iloc[0]['c']
        gold_count = client.query("SELECT COUNT(*) c FROM gold.fact_calls").to_dataframe().iloc[0]['c']

        if abs(silver_count - gold_count) > silver_count * 0.05:  # >5% difference
            raise ValueError(f"DATA QUALITY WARNING: Silver={silver_count}, Gold={gold_count}")

        print(f"Quality check passed. Silver={silver_count}, Gold={gold_count}, Dupes={dupes}")

    quality_check = PythonOperator(
        task_id='quality_check',
        python_callable=verify_data_quality,
    )

    # Pipeline order: load → clean → model → load facts → verify
    [load_va_calls, load_orders] >> build_silver >> build_dimensions >> build_facts >> quality_check
```

### Console UI: Cloud Composer Setup

1. Go to [console.cloud.google.com/composer](https://console.cloud.google.com/composer)
2. **Create Environment** → name: `call-center-pipeline` → location: `us-central1` → image version: latest → **Create** (takes ~20 min to provision)
3. Once running, click **Open Airflow UI** → upload the DAG file → it appears in the DAG list
4. Click the DAG → **Trigger DAG** to run manually, or wait for the 3 AM schedule

### Console UI: Monitoring the Pipeline

1. **Airflow UI** → DAG → click a run → see which tasks succeeded/failed, execution time, logs
2. **BigQuery** → query history shows every SQL run, bytes scanned, errors
3. **Cloud Monitoring** → set alerts: "if quality_check task fails, send email/Slack notification"

---

## GCP Service Summary

| Pipeline Step | GCP Service | What It Does | Equivalent on AWS |
|:---|:---|:---|:---|
| **Ingest (webhook)** | Cloud Functions | Receives webhook, writes to GCS | Lambda |
| **Ingest (scheduled)** | Cloud Scheduler + Cloud Functions | Triggers extract on a cron schedule | EventBridge + Lambda |
| **Storage (Bronze)** | GCS (Google Cloud Storage) | Raw file storage — the data lake | S3 |
| **Transform (Silver/Gold)** | BigQuery SQL | Dedup, clean, model — all in SQL | Athena / Redshift SQL, or Glue |
| **Transform (heavy)** | Dataflow (Apache Beam) | For streaming or complex transforms beyond SQL | AWS Glue / EMR |
| **Warehouse** | BigQuery | Serverless query engine — the star schema lives here | Redshift / Athena |
| **Orchestration** | Cloud Composer (Airflow) | Schedules and monitors the pipeline DAG | MWAA (Managed Airflow) / Step Functions |
| **Dashboard** | Looker Studio | Visual reports connected to BigQuery | QuickSight |
| **API** | Cloud Run | Serves query results via REST API | Lambda + API Gateway / ECS |
| **Monitoring** | Cloud Monitoring + Logging | Alerts on failures, latency, errors | CloudWatch |
| **IAM** | IAM | Who can access what | IAM (same concept) |

---

## The Complete Pipeline — One Picture

```
[Source Systems]
     │
     ▼
[Cloud Function / Cloud Scheduler]  ← INGEST
     │
     ▼
[GCS Bronze Bucket]                 ← RAW STORAGE
     │
     ▼
[BigQuery: raw dataset]             ← LOAD
     │
     ▼
[BigQuery: silver dataset]          ← CLEAN (dedup, timezone, type fix)
     │
     ▼
[BigQuery: gold dataset]            ← MODEL (dimensions + facts)
     │
     ├──→ [Looker Studio Dashboard]  ← SERVE (business users)
     ├──→ [Cloud Run API]            ← SERVE (applications)
     └──→ [ML Feature Query]         ← SERVE (AI pipeline)

[Cloud Composer DAG]                ← ORCHESTRATE (schedule + monitor + quality gates)
```

---

**Previous:** [03 — Building It](03_Building_It.md) — Build the star schema on BigQuery step by step.
**Related:** [02a — Source Tables](02a_Source_Tables.md) — What the source tables look like before this pipeline.
