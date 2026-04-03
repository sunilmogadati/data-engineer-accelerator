# Star Schema Design — Building It

**Step-by-step: build the star schema on BigQuery from raw call center data.**

---

## Prerequisites

- GCP project with billing enabled and BigQuery API active
- Call center data loaded into BigQuery (the `call_center_raw` dataset from the GCP Pipeline notebook)
- If not loaded yet, run the [GCP Pipeline notebook](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/GCP_Pipeline.ipynb) first

All SQL below runs in the [BigQuery console](https://console.cloud.google.com/bigquery) or via `bq query --use_legacy_sql=false`.

---

## Step 0: Create the Gold Dataset

The raw data is in `call_center_raw` (Bronze/Silver). The star schema lives in a separate dataset — the Gold layer.

```sql
-- Create a new dataset for the star schema
CREATE SCHEMA IF NOT EXISTS call_center_gold
OPTIONS (location = 'us-central1');
```

### Console UI
1. In BigQuery, click the three dots (⋮) next to your project name
2. **Create Dataset** → name: `call_center_gold` → location: `us-central1` → **Create**

---

## Step 1: dim_date — The Calendar Dimension

Every report filters by date. Instead of parsing timestamps in every query, `dim_date` pre-computes everything: day name, weekend flag, month, quarter, fiscal week.

**This is where the timezone conversion happens — ONCE.**

```sql
CREATE OR REPLACE TABLE call_center_gold.dim_date AS
WITH date_range AS (
    -- Generate one row per day for the full year
    SELECT d AS full_date
    FROM UNNEST(GENERATE_DATE_ARRAY('2026-01-01', '2026-12-31')) AS d
)
SELECT
    -- The key: YYYYMMDD as integer. Fast to join, human-readable.
    CAST(FORMAT_DATE('%Y%m%d', full_date) AS INT64)  AS date_key,

    full_date,

    -- Day attributes
    FORMAT_DATE('%A', full_date)                      AS day_name,        -- Monday, Tuesday, ...
    EXTRACT(DAYOFWEEK FROM full_date)                 AS day_of_week_num, -- 1=Sunday, 7=Saturday
    CASE WHEN EXTRACT(DAYOFWEEK FROM full_date) IN (1, 7)
         THEN TRUE ELSE FALSE END                     AS is_weekend,

    -- Week (Monday-start, matching LT reporting)
    DATE_TRUNC(full_date, ISOWEEK)                    AS monday_week,

    -- Month
    EXTRACT(MONTH FROM full_date)                     AS month_num,
    FORMAT_DATE('%B', full_date)                       AS month_name,     -- January, February, ...

    -- Quarter / Year
    EXTRACT(QUARTER FROM full_date)                   AS quarter,
    EXTRACT(YEAR FROM full_date)                      AS year

FROM date_range;

-- Verify
SELECT * FROM call_center_gold.dim_date
WHERE full_date BETWEEN '2026-03-01' AND '2026-03-07'
ORDER BY full_date;
```

**What this replaces in the stored proc:**
- `SET @fromDate = @fromDate AT TIME ZONE 'Eastern Standard Time' AT TIME ZONE 'UTC'` — timezone is handled by the pipeline populating this table
- `DATEPART(HOUR, DATEADD(HOUR, -4, u.StartedAt))` — no more manual `-4` hour offsets
- `CONVERT(CHAR(10), CAST(...))` — date formatting is pre-computed

### Console UI
Run the SQL above in the BigQuery query editor. Then click `call_center_gold` → `dim_date` → **Preview** to verify.

---

## Step 2: dim_time — Hourly Periods

The flat table has 96 columns for half-hour breakdowns (00:00 calls, 00:30 calls, 01:00 calls, ...). A dimension table replaces all of them.

```sql
CREATE OR REPLACE TABLE call_center_gold.dim_time AS
SELECT
    hour                                              AS time_key,
    hour,

    -- Half-hour label
    CONCAT(LPAD(CAST(hour AS STRING), 2, '0'), ':00') AS half_hour_start,
    CONCAT(LPAD(CAST(hour AS STRING), 2, '0'), ':30') AS half_hour_end,

    -- Period of day
    CASE
        WHEN hour BETWEEN 6 AND 11  THEN 'Morning'
        WHEN hour BETWEEN 12 AND 17 THEN 'Afternoon'
        WHEN hour BETWEEN 18 AND 21 THEN 'Evening'
        ELSE 'Night'
    END                                               AS period

FROM UNNEST(GENERATE_ARRAY(0, 23)) AS hour;

-- Verify
SELECT * FROM call_center_gold.dim_time ORDER BY time_key;
```

**What this replaces:** 96 columns in the flat table (`00`, `00:00`, `00:30`, `01`, `01:00`, `01:30`, ..., `23:30`). Now a single `GROUP BY dt.period` or `GROUP BY dt.hour` gives any time breakdown.

---

## Step 3: dim_campaign — Client/Program/DNIS Mapping

This replaces the hard-coded program name overrides and the DNIS→campaign lookup that is scattered through the stored proc.

```sql
CREATE OR REPLACE TABLE call_center_gold.dim_campaign AS
SELECT
    ROW_NUMBER() OVER (ORDER BY dnis) AS campaign_key,
    CAST(dnis AS STRING)              AS dnis,
    campaign_name,
    client_name,
    channel,
    start_date,
    end_date
FROM call_center_raw.campaigns;

-- Verify
SELECT * FROM call_center_gold.dim_campaign;
```

**What this replaces:**
- `UPDATE t SET t.programName = 'Bullseye Pro English' WHERE t.client = 'Emson'` — the correct name is a column in this table
- `WHERE t.id IN (1094758, 1085870, 1090805)` — hard-coded call ID overrides disappear. The campaign mapping is data.

> **Note:** The synthetic dataset has a simple campaign table. In production, this dimension would also include `source_name`, `media_source` (buyer), `media_type`, `buyer_source_code`, `script_id` — all from `tblSource`. The principle is the same: put the mapping in a dimension table, not in a stored proc.

---

## Step 4: dim_disposition — Disposition Lookup

This replaces the 50 lines of CASE WHEN that map FailReason + Status codes to human-readable disposition names.

```sql
CREATE OR REPLACE TABLE call_center_gold.dim_disposition AS
SELECT
    ROW_NUMBER() OVER (ORDER BY disposition) AS disposition_key,
    disposition                               AS disposition_name,
    disposition                               AS disposition_type,

    -- Boolean flags for easy filtering
    CASE WHEN LOWER(disposition) = 'completed' THEN TRUE ELSE FALSE END AS is_sale,
    CASE WHEN LOWER(disposition) IN ('voicemail', 'abandoned') THEN TRUE ELSE FALSE END AS is_abandon,
    CASE WHEN LOWER(disposition) IN ('callback') THEN TRUE ELSE FALSE END AS is_callback

FROM (
    SELECT DISTINCT disposition
    FROM call_center_raw.calls
    WHERE disposition IS NOT NULL
);

-- Verify
SELECT * FROM call_center_gold.dim_disposition;
```

**What this replaces in production:** The 50-line CASE WHEN block mapping `FailReason + status` → `DispoType + DispoName`. In the star schema, each combination is a row in this table. Adding a new disposition type = INSERT a row, not modify the stored proc.

> **Production version** would have columns: `fail_reason` (INT), `status` (INT), `disposition_type` (the internal code like `eu_abandoned`), `disposition_name` (the display name like `ABANDON`). The pipeline matches incoming calls to the right disposition row using `fail_reason + status` as the lookup key.

---

## Step 5: fact_calls — The Fact Table

One row per call. Surrogate keys point to dimensions. Measures (duration, revenue) are the numbers being analyzed.

```sql
CREATE OR REPLACE TABLE call_center_gold.fact_calls
PARTITION BY call_date
CLUSTER BY campaign_key
AS
SELECT
    -- Surrogate key
    ROW_NUMBER() OVER (ORDER BY c.call_id)   AS call_key,

    -- Dimension keys
    CAST(FORMAT_DATE('%Y%m%d', c.date) AS INT64) AS date_key,
    EXTRACT(HOUR FROM c.start_time)               AS time_key,
    dc.campaign_key,
    dd.disposition_key,

    -- Source reference (for tracing back to original systems)
    c.call_id,
    c.channel                                      AS call_type,
    c.date                                         AS call_date,

    -- Measures
    c.duration                                     AS duration_sec,

    -- Order data (LEFT JOIN — not every call has an order)
    o.order_id,
    o.total                                        AS order_total,
    CASE WHEN o.order_id IS NOT NULL THEN TRUE ELSE FALSE END AS is_order,

    -- Caller info
    CAST(c.dnis AS STRING)                         AS dnis,
    c.caller_phone                                 AS caller_ani

FROM call_center_raw.calls c

-- Join to get campaign key
LEFT JOIN call_center_gold.dim_campaign dc
    ON CAST(c.dnis AS STRING) = dc.dnis

-- Join to get disposition key
LEFT JOIN call_center_gold.dim_disposition dd
    ON LOWER(c.disposition) = LOWER(dd.disposition_name)

-- Join to get order data
LEFT JOIN call_center_raw.orders o
    ON c.call_id = o.call_id;

-- Verify
SELECT COUNT(*) AS total_calls,
       COUNTIF(is_order) AS orders,
       ROUND(COUNTIF(is_order) / COUNT(*) * 100, 1) AS conversion_pct
FROM call_center_gold.fact_calls;
```

**What this replaces:**
- The entire 891-line stored proc's final output
- 315 columns → ~15 columns (dimensions joined at query time, not pre-joined)
- `PARTITION BY call_date` — BigQuery only scans relevant date partitions, not the full table
- `CLUSTER BY campaign_key` — queries filtered by campaign skip irrelevant data blocks

**Dedup:** The Silver layer (before this step) deduplicates by `call_id`. The fact table has `ROW_NUMBER()` generating a unique `call_key`. Duplicates are impossible by design.

---

## Step 6: Query the Star Schema

Now run the same reports that the flat table supports — but with simple, readable SQL.

### Report 1: Campaign Performance

```sql
-- Campaign conversion rate, revenue, average duration
-- In the flat table: this requires knowing which of the 315 columns to use
-- In the star schema: a clear join pattern

SELECT
    dc.client_name,
    dc.campaign_name,
    dc.channel,
    COUNT(*)                          AS total_calls,
    COUNTIF(f.is_order)               AS orders,
    ROUND(COUNTIF(f.is_order) / COUNT(*) * 100, 1) AS conversion_pct,
    ROUND(SUM(f.order_total), 2)      AS total_revenue,
    ROUND(AVG(f.duration_sec), 1)     AS avg_duration_sec
FROM call_center_gold.fact_calls f
JOIN call_center_gold.dim_campaign dc ON f.campaign_key = dc.campaign_key
GROUP BY dc.client_name, dc.campaign_name, dc.channel
ORDER BY total_revenue DESC;
```

### Report 2: Hourly Volume (Staffing)

```sql
-- Calls per hour — when do we need more agents?

SELECT
    dt.hour,
    dt.period,
    COUNT(*)              AS total_calls,
    COUNTIF(f.is_order)   AS orders,
    ROUND(AVG(f.duration_sec), 1) AS avg_duration
FROM call_center_gold.fact_calls f
JOIN call_center_gold.dim_time dt ON f.time_key = dt.time_key
GROUP BY dt.hour, dt.period
ORDER BY dt.hour;
```

### Report 3: Day-of-Week Pattern

```sql
-- Which days have the most calls? The most orders?

SELECT
    dd.day_name,
    dd.is_weekend,
    COUNT(*)              AS total_calls,
    COUNTIF(f.is_order)   AS orders,
    ROUND(COUNTIF(f.is_order) / COUNT(*) * 100, 1) AS conversion_pct
FROM call_center_gold.fact_calls f
JOIN call_center_gold.dim_date dd ON f.date_key = dd.date_key
GROUP BY dd.day_name, dd.is_weekend
ORDER BY total_calls DESC;
```

### Report 4: Disposition Breakdown

```sql
-- Why are calls ending the way they do?

SELECT
    di.disposition_name,
    di.is_sale,
    di.is_abandon,
    COUNT(*) AS call_count,
    ROUND(COUNT(*) / SUM(COUNT(*)) OVER () * 100, 1) AS pct_of_total
FROM call_center_gold.fact_calls f
JOIN call_center_gold.dim_disposition di ON f.disposition_key = di.disposition_key
GROUP BY di.disposition_name, di.is_sale, di.is_abandon
ORDER BY call_count DESC;
```

### Report 5: The VP's 3 AM Question

```sql
-- "Show me conversion rate by campaign, by day of week, by time of day, for March"
-- On the flat table: complex, error-prone, timezone issues
-- On the star schema: one query, clear, correct

SELECT
    dc.campaign_name,
    dc.channel,
    dd.day_name,
    dt.period,
    COUNT(*)              AS total_calls,
    COUNTIF(f.is_order)   AS orders,
    ROUND(COUNTIF(f.is_order) / COUNT(*) * 100, 1) AS conversion_pct
FROM call_center_gold.fact_calls f
JOIN call_center_gold.dim_campaign dc ON f.campaign_key = dc.campaign_key
JOIN call_center_gold.dim_date dd ON f.date_key = dd.date_key
JOIN call_center_gold.dim_time dt ON f.time_key = dt.time_key
WHERE dd.month_name = 'March'
GROUP BY dc.campaign_name, dc.channel, dd.day_name, dt.period
ORDER BY dc.campaign_name, conversion_pct DESC;
```

---

## Step 7: Data Quality Check

The star schema makes data quality visible. Run these after every pipeline run.

```sql
-- Check 1: Any calls without a campaign match? (DNIS not in dim_campaign)
SELECT COUNT(*) AS orphan_calls
FROM call_center_gold.fact_calls
WHERE campaign_key IS NULL;

-- Check 2: Any calls without a disposition match?
SELECT COUNT(*) AS no_disposition
FROM call_center_gold.fact_calls
WHERE disposition_key IS NULL;

-- Check 3: Duplicate call_ids? (should be zero)
SELECT call_id, COUNT(*) AS dupes
FROM call_center_gold.fact_calls
GROUP BY call_id
HAVING COUNT(*) > 1;

-- Check 4: Orders without matching calls?
SELECT COUNT(*) AS orphan_orders
FROM call_center_raw.orders o
LEFT JOIN call_center_gold.fact_calls f ON o.call_id = f.call_id
WHERE f.call_key IS NULL;
```

If any check returns non-zero results, the problem is in the pipeline — not in 891 lines of tangled SQL. And the check tells exactly WHERE: missing DNIS mapping? Missing disposition? Duplicate in source? Orphaned order?

---

## What Was Built

| Layer | What It Contains | Row Count |
|:---|:---|:---|
| `call_center_raw` (Bronze) | Raw tables as loaded from GCS | Source counts |
| `call_center_gold.dim_date` | Calendar for 2026 | 365 |
| `call_center_gold.dim_time` | Hours 0-23 with periods | 24 |
| `call_center_gold.dim_campaign` | Campaign/DNIS mapping | ~10 |
| `call_center_gold.dim_disposition` | Disposition lookup | ~6-8 |
| `call_center_gold.fact_calls` | One row per call, partitioned by date | ~500 |

**Total SQL to build it:** ~100 lines of CREATE TABLE statements. Each one is independent, testable, and understandable.

**Total SQL it replaces:** 891 lines of tangled stored procedure logic.

---

## What Comes Next

1. **Automate the pipeline** — Cloud Composer (Airflow) runs these CREATE TABLE statements nightly. New data lands in Bronze → pipeline rebuilds Silver and Gold automatically.
2. **Add the Silver layer explicitly** — right now, Gold reads directly from Bronze. In production, Silver handles dedup, timezone conversion, null flagging — and Gold reads from Silver.
3. **Connect a dashboard** — Looker Studio reads from the Gold tables. The VP sees the reports without writing SQL.
4. **Add more dimensions** — `dim_agent` (when agent data is available), `dim_product` (for product-level revenue analysis).
5. **Add fact_agent_activity** — separate fact table for agent utilization, replacing the CROSS JOIN expansion in the stored proc.
