# Data Modeling — Hello World

**Build the call center star schema on BigQuery. See the difference a data model makes.**

> **Prerequisites:** A GCP (Google Cloud Platform) account with BigQuery enabled. If the account is not set up yet, see the GCP Setup section in the Cloud Pipeline notebook (M08, Section 3.1a). Basic SQL — `SELECT`, `JOIN`, `GROUP BY`, `WHERE`. If SQL syntax is unfamiliar, the [SQL Foundations notebook](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M02_Git_Linux_SQL_Foundations.ipynb) covers everything needed.

---

## What This Demonstrates

In the next 15 minutes:
1. Upload raw call center CSV files to **GCS (Google Cloud Storage, pronounced "G-C-S")** — Google's equivalent of AWS S3
2. Load them into **BigQuery (pronounced "big KWEER-ee")** — Google's serverless data warehouse
3. Query the raw data — and see why a flat table is painful
4. Build dimension tables and a fact table — the star schema
5. Query the star schema — and see the same question answered in one clean query

The "before and after" is the point. Same data. Same question. Wildly different experience.

---

## Step 1: Upload Raw Data to GCS

```bash
# Set your project (replace with your GCP project ID)
gcloud config set project YOUR_PROJECT_ID

# Create a bucket (bucket names are globally unique — add your initials or a random suffix)
gsutil mb -l us-central1 gs://de-accelerator-YOUR_NAME/

# Upload the call center data files
gsutil cp data/calls.json gs://de-accelerator-YOUR_NAME/bronze/
gsutil cp data/campaigns.csv gs://de-accelerator-YOUR_NAME/bronze/
gsutil cp data/orders.csv gs://de-accelerator-YOUR_NAME/bronze/
gsutil cp data/products.csv gs://de-accelerator-YOUR_NAME/bronze/
gsutil cp data/payments.xml gs://de-accelerator-YOUR_NAME/bronze/

# Verify
gsutil ls gs://de-accelerator-YOUR_NAME/bronze/
```

**Expected output:**
```
gs://de-accelerator-YOUR_NAME/bronze/calls.json
gs://de-accelerator-YOUR_NAME/bronze/campaigns.csv
gs://de-accelerator-YOUR_NAME/bronze/orders.csv
gs://de-accelerator-YOUR_NAME/bronze/products.csv
gs://de-accelerator-YOUR_NAME/bronze/payments.xml
```

> **Bronze** is the raw data layer — files land here exactly as they arrived. No cleaning, no transformation. This is the "Bronze → Silver → Gold" medallion pattern covered in the Cloud Pipeline material.

---

## Step 2: Create a BigQuery Dataset and Load Tables

```bash
# Create a BigQuery dataset (a container for tables — like a schema in PostgreSQL)
bq mk --dataset --location=us-central1 YOUR_PROJECT_ID:call_center_raw

# Load campaigns CSV into BigQuery (auto-detect schema)
bq load --autodetect --source_format=CSV \
  call_center_raw.campaigns \
  gs://de-accelerator-YOUR_NAME/bronze/campaigns.csv

# Load products CSV
bq load --autodetect --source_format=CSV \
  call_center_raw.products \
  gs://de-accelerator-YOUR_NAME/bronze/products.csv

# Load orders CSV
bq load --autodetect --source_format=CSV \
  call_center_raw.orders \
  gs://de-accelerator-YOUR_NAME/bronze/orders.csv

# Load calls JSON (newline-delimited JSON)
bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON \
  call_center_raw.calls \
  gs://de-accelerator-YOUR_NAME/bronze/calls.json
```

**Verify in the BigQuery console** (console.cloud.google.com/bigquery):
- The `call_center_raw` dataset should appear in the left panel
- Click each table → Preview tab → confirm data loaded

---

## Step 3: Query the Raw Data — See the Problem

Open the BigQuery console query editor and run:

```sql
-- "What is the average wait time by campaign for March?"
-- On the RAW table — no data model

SELECT
    c.campaign_name,
    AVG(calls.wait_time) AS avg_wait_seconds,
    COUNT(*) AS total_calls
FROM call_center_raw.calls AS calls
JOIN call_center_raw.campaigns AS c
    ON calls.dnis = c.dnis
WHERE calls.call_date >= '2026-03-01'
GROUP BY c.campaign_name
ORDER BY avg_wait_seconds DESC;
```

This works — but notice:
- The join is on `dnis` (a phone number string) — slow string comparison
- `campaign_name` comes from a separate table every time — if the name has typos in either table, the join silently drops rows
- Adding "by day of week" requires `EXTRACT(DAYOFWEEK FROM call_date)` — computed at query time
- Adding "by time of day" requires more extraction logic — getting messy
- No history tracking — if a campaign switches from VA to Live Agent mid-month, the old channel value is overwritten

Now imagine this on 50 million rows instead of 510. Every added dimension makes the query more complex and slower.

---

## Step 4: Build the Star Schema

```sql
-- Create the analytical dataset
CREATE SCHEMA IF NOT EXISTS call_center_star;

-- ============================================================
-- DIMENSION: dim_date
-- Pre-computed calendar. Analysts filter by day name, weekend,
-- month, quarter — without parsing timestamps at query time.
-- ============================================================
CREATE OR REPLACE TABLE call_center_star.dim_date AS
SELECT
    CAST(FORMAT_DATE('%Y%m%d', d) AS INT64) AS date_key,
    d AS full_date,
    FORMAT_DATE('%A', d) AS day_name,        -- Monday, Tuesday, ...
    EXTRACT(MONTH FROM d) AS month_num,
    FORMAT_DATE('%B', d) AS month_name,      -- January, February, ...
    EXTRACT(QUARTER FROM d) AS quarter,
    EXTRACT(YEAR FROM d) AS year,
    CASE WHEN EXTRACT(DAYOFWEEK FROM d) IN (1, 7) THEN TRUE ELSE FALSE END AS is_weekend
FROM UNNEST(
    GENERATE_DATE_ARRAY('2026-01-01', '2026-12-31')
) AS d;

-- ============================================================
-- DIMENSION: dim_time
-- Hourly periods so analysts can group by Morning/Afternoon/Evening.
-- ============================================================
CREATE OR REPLACE TABLE call_center_star.dim_time AS
SELECT
    hour AS time_key,
    hour,
    CASE
        WHEN hour BETWEEN 6 AND 11 THEN 'Morning'
        WHEN hour BETWEEN 12 AND 17 THEN 'Afternoon'
        WHEN hour BETWEEN 18 AND 21 THEN 'Evening'
        ELSE 'Night'
    END AS period
FROM UNNEST(GENERATE_ARRAY(0, 23)) AS hour;

-- ============================================================
-- DIMENSION: dim_campaign
-- One row per campaign. Surrogate key for fast joins.
-- ============================================================
CREATE OR REPLACE TABLE call_center_star.dim_campaign AS
SELECT
    ROW_NUMBER() OVER (ORDER BY dnis) AS campaign_key,
    dnis,
    campaign_name,
    client_name,
    channel,
    start_date,
    end_date
FROM call_center_raw.campaigns;

-- ============================================================
-- DIMENSION: dim_product
-- One row per product. Surrogate key.
-- ============================================================
CREATE OR REPLACE TABLE call_center_star.dim_product AS
SELECT
    ROW_NUMBER() OVER (ORDER BY sku) AS product_key,
    sku,
    description,
    base_price,
    category,
    is_upsell,
    is_upgrade
FROM call_center_raw.products;

-- ============================================================
-- FACT TABLE: fact_calls
-- One row per call. Surrogate keys point to dimensions.
-- Measures: duration, wait_time, outcome.
-- ============================================================
CREATE OR REPLACE TABLE call_center_star.fact_calls
PARTITION BY call_date
CLUSTER BY campaign_key
AS
SELECT
    calls.call_id AS call_key,
    CAST(FORMAT_DATE('%Y%m%d', calls.call_date) AS INT64) AS date_key,
    EXTRACT(HOUR FROM calls.call_timestamp) AS time_key,
    dc.campaign_key,
    calls.duration AS duration_seconds,
    calls.wait_time AS wait_time_seconds,
    calls.outcome
FROM call_center_raw.calls AS calls
JOIN call_center_star.dim_campaign AS dc
    ON calls.dnis = dc.dnis;
```

> **Note:** The exact column names may differ from your raw data. Adjust the `calls.call_date`, `calls.call_timestamp`, `calls.duration`, `calls.wait_time`, and `calls.outcome` references to match your actual schema. Run `SELECT * FROM call_center_raw.calls LIMIT 5` first to see the column names.

---

## Step 5: Query the Star Schema — See the Difference

```sql
-- Same question: "Average wait time by campaign for March"
-- Now with the star schema:

SELECT
    dc.campaign_name,
    dc.channel,
    AVG(f.wait_time_seconds) AS avg_wait,
    COUNT(*) AS total_calls
FROM call_center_star.fact_calls AS f
JOIN call_center_star.dim_campaign AS dc ON f.campaign_key = dc.campaign_key
JOIN call_center_star.dim_date AS dd ON f.date_key = dd.date_key
WHERE dd.month_name = 'March'
GROUP BY dc.campaign_name, dc.channel
ORDER BY avg_wait DESC;
```

**What changed:**
- Join on integer keys (campaign_key, date_key) — fast
- Filter by `month_name = 'March'` — no date parsing at query time
- Channel is already a column in dim_campaign — no extra logic
- Adding "by day of week" is just adding `dd.day_name` to SELECT and GROUP BY — one column, already computed
- Adding "by time of day" is joining dim_time and using `dt.period` — no EXTRACT, no CASE statement

```sql
-- The 3 AM query: "Wait time by campaign, by day of week, by time of day, for March"
-- On the star schema — clean, readable, fast:

SELECT
    dc.campaign_name,
    dc.channel,
    dd.day_name,
    dt.period,
    AVG(f.wait_time_seconds) AS avg_wait,
    COUNT(*) AS total_calls
FROM call_center_star.fact_calls AS f
JOIN call_center_star.dim_campaign AS dc ON f.campaign_key = dc.campaign_key
JOIN call_center_star.dim_date AS dd ON f.date_key = dd.date_key
JOIN call_center_star.dim_time AS dt ON f.time_key = dt.time_key
WHERE dd.month_name = 'March'
GROUP BY dc.campaign_name, dc.channel, dd.day_name, dt.period
ORDER BY dc.campaign_name, avg_wait DESC;
```

That is the VP's 3 AM question — answered in one readable SQL query, running in seconds on BigQuery. The data model made it possible.

---

## What Just Happened

| Step | What It Did | AWS Equivalent |
|:---|:---|:---|
| Upload to GCS | Raw files in cloud storage | `aws s3 cp` to S3 |
| Create BigQuery dataset | Container for tables | Create Redshift schema or Athena database |
| `bq load` | Load CSV/JSON into BigQuery tables | `COPY` into Redshift or Glue Crawler on S3 |
| Create dimension tables | Pre-computed lookup tables with surrogate keys | Same SQL, different warehouse |
| Create fact table with PARTITION + CLUSTER | Partitioned by date, clustered by campaign | Redshift DISTKEY/SORTKEY or Athena partitioned tables |
| Query the star schema | Integer joins, pre-computed attributes, clean GROUP BY | Same SQL — the star schema pattern is universal |

> **The GCP-specific parts:** GCS for storage, BigQuery for warehouse, `bq` CLI for loading, `PARTITION BY` and `CLUSTER BY` syntax. **The data modeling parts are identical to AWS** — star schema, surrogate keys, fact/dimension split. The architecture transfers. Only the CLI and syntax change.

---

## Try It Yourself

1. **Run every query above** in the BigQuery console. See the results.
2. **Add a new dimension:** Build `dim_agent` from the calls data (extract unique agent names). Add `agent_key` to the fact table. Now query "average handle time by agent by campaign."
3. **Break it:** What happens if you join fact_calls to dim_campaign on the natural key (dnis string) instead of campaign_key (integer)? Does it still work? Is it slower?
4. **Ask a question the raw table cannot answer easily:** "Which campaign had the highest weekend call volume in March, broken down by Morning/Afternoon/Evening?" On the star schema, this is one GROUP BY with 3 joins. On the raw table, it requires date extraction, hour extraction, CASE statements for periods, weekend logic — messy.

---

## What Comes Next

| Tomorrow's Topic | Where It Leads |
|:---|:---|
| **Bronze → Silver → Gold pipeline** | Automate the transformation from raw CSVs to the star schema. Not manual SQL — a repeatable pipeline. |
| **Cloud Composer (Airflow)** | Schedule the pipeline to run every night. New data lands in GCS → triggers Bronze → Silver → Gold → star schema refresh. |
| **Data quality checks** | The raw data has intentional issues (dupes, nulls, timezone bugs). The Silver layer cleans them. How? |

---

**Hands-on notebook:** [M05 — Data Modeling](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M05_Data_Modeling.ipynb) — the same star schema built step-by-step with Python + SQL, including SCD Type 2, edge cases, and interview questions.

**Full pipeline build:** [M08 — Cloud Data Pipeline Build](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M08_Cloud_Data_Pipeline_Build.ipynb) — 208 cells covering GCS, BigQuery, Dataflow, Bronze/Silver/Gold, and 20 GCP subsections with console walkthroughs.
