# Microsoft Fabric: News Analytics Pipeline

> Complete end-to-end data analytics pipeline built in Microsoft Fabric. Ingests news headlines from NewsAPI.org, applies medallion-architecture transformations, scores sentiment with VADER, orchestrates everything on a schedule, alerts on failures, and notifies on success via Teams.

**Author:** Mustafa Abdeali
**Workspace:** `news-pipeline`
**Built:** May 12–13, 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Build Journey](#3-build-journey)
4. [Issues Encountered & Solutions](#4-issues-encountered--solutions)
5. [Key Lessons Learned](#5-key-lessons-learned)
6. [Operations Guide](#6-operations-guide)
7. [Next Steps](#7-next-steps)
8. [Glossary](#8-glossary)

---

## 1. Overview

End-to-end data analytics pipeline demonstrating production data engineering patterns inside Microsoft Fabric.

### What's Inside

- **Bronze layer** — raw news headlines from NewsAPI.org with permanent ingestion timestamps
- **Daily summary** — aggregated article counts per day, accumulating history
- **Silver layer** — cleaned, categorized, deduplicated headlines with extracted domains
- **Sentiment layer** — VADER-scored headlines with sentiment labels (Positive/Neutral/Negative)
- **Power BI dashboard** — interactive visualizations with cross-filtering, slicers, sentiment color-coding
- **Email alerts** — Office 365 Outlook notifications when any pipeline step fails
- **Teams notifications** — success messages with dashboard link after each complete run

### Project Scale

| Component | Count |
|---|---|
| Notebooks | 4 (`fetch_news`, `daily_summary`, `silver_enrichment`, `sentiment_enrichment`) |
| Delta tables | 4 (`top_headlines`, `daily_ingestion_summary`, `top_headlines_silver`, `top_headlines_sentiment`) |
| Dashboard visuals | 9+ (slicers, pie, donut, bar, line, table) |
| Pipeline activities | 9 (4 notebooks + 4 failure emails + 1 Teams success) |
| Run cadence | Every 6 hours, automated |

> **Why this matters** — This is not just a tutorial pipeline. It uses the same patterns real data engineering teams use: idempotent ingestion, medallion architecture, scheduled orchestration, failure alerting, and BI delivery.

---

## 2. Architecture

### Data Flow

```
NewsAPI.org (https://newsapi.org/v2/top-headlines)
        |
        v   every 6 hours, via scheduled pipeline
+----------------------------------------------+
| BRONZE: top_headlines                        |
| Raw API response, MERGE on URL               |
| ingested_at = permanent (set on insert only) |
+----------------------------------------------+
        |
        v
+----------------------------------------------+
| AGGREGATE: daily_ingestion_summary           |
| One row per day, MERGE keyed on day          |
| Accumulates history across runs              |
+----------------------------------------------+
        |
        v
+----------------------------------------------+
| SILVER: top_headlines_silver                 |
| Junk filtered, domain extracted via regex,   |
| category assigned by keyword matching,       |
| timestamp parsed, duplicates removed         |
+----------------------------------------------+
        |
        v
+----------------------------------------------+
| SENTIMENT: top_headlines_sentiment           |
| VADER scoring via UDF on title column        |
| sentiment_compound + label (Pos/Neu/Neg)     |
+----------------------------------------------+
        |
        v
+----------------------------------------------+
| SEMANTIC MODEL: news_model_full              |
| DirectLake mode (no manual refresh)          |
+----------------------------------------------+
        |
        v
+----------------------------------------------+
| DASHBOARD: News Dashboard v3                 |
| Interactive Power BI report                  |
+----------------------------------------------+
```

### Orchestration & Alerting

```
fetch_news --✓--> daily_summary --✓--> silver_enrichment --✓--> sentiment_enrichment
    |                 |                     |                         |
    v ✗               v ✗                   v ✗                       v ✗  ↘ ✓
  [Email1]         [Email2]              [Email3]                 [Email4] [Teams]
```

---

## 3. Build Journey

### Phase 1 — Bronze Layer (Ingestion)

**Goal:** Pull raw headlines from NewsAPI.org into a Lakehouse Delta table with URL-based deduplication.

**Steps:**
1. Created Lakehouse `news_lakehouse` in workspace `news-pipeline`
2. Registered for NewsAPI.org, generated API key
3. Created notebook `fetch_news`, attached `news_lakehouse` as default
4. Cell 1: Store API key (later: move to Azure Key Vault)
5. Cell 2: Call NewsAPI top-headlines endpoint, parse JSON
6. Cell 3: Build Spark DataFrame with `last_seen_at` timestamp
7. Cell 4: MERGE into `top_headlines` — insert new URLs, update `last_seen_at` on matches

**Key pattern — the bulletproof MERGE:**

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import current_timestamp

table_name = "top_headlines"

try:
    delta_tbl = DeltaTable.forName(spark, table_name)
    (delta_tbl.alias("t")
        .merge(new_df.alias("s"), "t.url = s.url")
        .whenNotMatchedInsert(values={
            "source_name":  "s.source_name",
            "title":        "s.title",
            "url":          "s.url",
            "published_at": "s.published_at",
            "ingested_at":  "current_timestamp()",  # set ONCE
            "last_seen_at": "s.last_seen_at"
        })
        .whenMatchedUpdate(set={
            "last_seen_at": "s.last_seen_at"  # only this updates
        })
        .execute())
except Exception as e:
    new_df.withColumn("ingested_at", current_timestamp()).write.saveAsTable(table_name)
```

> ⚠️ **Critical insight** — `ingested_at` is set only when a row is first inserted. On subsequent MERGE re-encounters, only `last_seen_at` updates. This guarantees historical accuracy even if the same article appears in 100 future API calls.

### Phase 2 — Daily Summary Aggregation

**Goal:** Maintain a separate table with one row per day for the dashboard's time-series chart.

**Steps:**
1. Created notebook `daily_summary`
2. Read `top_headlines`, group by `DATE(ingested_at)`, count rows
3. MERGE into `daily_ingestion_summary` keyed on `day`
4. Enabled schema auto-merge to allow adding `last_updated` column later

**Schema evolution setup:**

```python
spark.conf.set('spark.databricks.delta.schema.autoMerge.enabled', 'true')

# whenMatchedUpdateAll + whenNotMatchedInsertAll handles new columns gracefully
```

### Phase 3 — Power BI Dashboard

**Goal:** Interactive dashboard on top of the Lakehouse.

**Steps:**
1. Clicked **New semantic model** from the lakehouse → selected existing tables
2. From the semantic model → created a new report (blank canvas)
3. Added visuals: bar chart (sources), line chart (articles over time), table (recent headlines)
4. Saved as `News Dashboard`
5. Verified DirectLake storage mode (no manual refresh needed)

> 💡 **DirectLake explained** — DirectLake queries the Lakehouse directly. When the pipeline writes new data, the dashboard reflects it within seconds. No scheduled refresh required.

### Phase 4 — Pipeline Orchestration

**Goal:** Chain notebooks with success/failure handling and put on a schedule.

**Steps:**
1. Created Data Pipeline `news_pipeline_orchestrator`
2. Added Notebook activities for `fetch_news` and `daily_summary`
3. Connected with green ✓ success handles
4. Saved and ran manually to verify
5. Configured Schedule: every 6 hours, IST timezone, 30-day end date

### Phase 5 — Silver Layer (Enrichment)

**Goal:** Clean and enrich Bronze into a business-ready Silver table.

**The 5 enrichment steps:**

1. **Filter junk** — drop null/`[Removed]`/very short titles
2. **Parse timestamp** — `published_at` string → real timestamp via `to_timestamp()`
3. **Extract domain** — regex pulls `apnews.com` from full URL
4. **Categorize** — `when().rlike()` chain → Tech/Politics/Business/Sports/World/Other
5. **Deduplicate** — `row_number()` partition by URL, keep most recent

**Why overwrite instead of MERGE:** Silver is derived from Bronze. Can always regenerate. Overwrite is simpler and safer for derived layers.

### Phase 6 — Sentiment Analysis

**Goal:** Score each headline's sentiment using VADER.

**Steps:**
1. Created notebook `sentiment_enrichment`
2. Cell 1: `%pip install vaderSentiment`
3. Built UDF wrapping VADER's `polarity_scores()`
4. Applied UDF to `title` column
5. Bucketed compound: `>0.05` = Positive, `<-0.05` = Negative, else Neutral
6. Saved as `top_headlines_sentiment` (overwrite)

**Sample sentiment distribution observed:**

| Label | Count | % |
|---|---|---|
| Neutral | 10 | 55.6% |
| Negative | 5 | 27.8% |
| Positive | 3 | 16.7% |

### Phase 7 — Failure Alerting (Email)

**Goal:** Get an email immediately when any notebook fails.

**Steps:**
1. Added Office 365 Outlook activity for each notebook
2. Authenticated Outlook connector once (reused for all)
3. Configured To/Subject/Body with `@{pipeline().RunId}` and `@{utcnow()}` expressions
4. Wired the red ✗ failure handle to each email
5. Tested with `raise Exception(...)` — confirmed email arrived

### Phase 8 — Success Notifications (Teams)

**Goal:** Get a Teams notification with dashboard link after every successful run.

**Steps:**
1. Added Microsoft Teams activity at end of pipeline
2. Authenticated Teams connector
3. `Post in: Group chat` → self-chat for personal notifications
4. Composed message with run details + dashboard URL
5. Wired green ✓ success handle from `sentiment_enrichment` → Teams

> 💚 **Heartbeat pattern** — Alerts catch breakage. Heartbeats prove health. Together: you can distinguish "pipeline broke" from "pipeline silently stopped running."

---

## 4. Issues Encountered & Solutions

| Issue | Root Cause | Solution |
|---|---|---|
| `UnsupportedOperationException: No default context found` | Notebook hadn't been attached to a default Lakehouse | In notebook explorer panel, set `news_lakehouse` as default (📌 pin icon) |
| Pipeline silently overwriting Bronze every run | `tableExists()` check unreliable; else branch ran every time → `CREATE OR REPLACE TABLE` | Replaced with `try/except` on `DeltaTable.forName()`. Use explicit `whenNotMatchedInsert` with `current_timestamp()` for new rows; `whenMatchedUpdate` only touches `last_seen_at` |
| Yesterday's data appeared lost (only May 13 in line chart) | Each MERGE was re-touching all rows and refreshing `ingested_at` | Used Delta time travel: `spark.read.format('delta').option('versionAsOf', 4).table('top_headlines')` — recovered May 12 snapshot |
| Schema mismatch when adding `last_updated` column | Delta requires explicit permission to evolve schema | `spark.conf.set('spark.databricks.delta.schema.autoMerge.enabled', 'true')`. Use `whenMatchedUpdateAll + whenNotMatchedInsertAll` |
| New tables not appearing in semantic model | Default semantic models don't auto-sync | Recreated semantic model (`news_model_full` → `news_model_v3`) including all tables. Built new dashboard version |
| Card visual missing from Visualizations panel | Power BI UI variation in Fabric | Used Multi-row card with `Count` aggregation, or Pie chart for breakdown. Visual choice is flexible |
| `'DATE' is not a recognized built-in function` | SQL analytics endpoint uses T-SQL, not Spark SQL | T-SQL: `CAST(col AS DATE)`. Spark SQL (notebooks): `DATE(col)` |
| `Connection is required` on Outlook/Teams activities | First-time authentication needed | Click `+ New` on Connection dropdown, sign in with Microsoft 365 account. Reuse connection across activities |
| Slicer accidentally created instead of Card | Empty visual placeholders look similar | Use search box in Visualizations panel. Check data well: Slicer uses `Field`, Card uses `Values` |
| `TABLE_OR_VIEW_ALREADY_EXISTS` during MERGE fallback | Fallback tried `saveAsTable` without `overwrite` on existing table | Use `mode('overwrite').option('overwriteSchema', 'true')` in fallback |

---

## 5. Key Lessons Learned

### Idempotency is non-negotiable

Every notebook in a scheduled pipeline must be safe to re-run without corrupting data.
- Delta MERGE for incremental Bronze data
- Explicit `whenMatched`/`whenNotMatched` logic
- Overwrite mode for derived tables (Silver, Sentiment) since they regenerate from upstream

### Bronze is sacred

Once data enters Bronze, never modify or delete. Use immutable timestamps (`ingested_at` set only on insert). Bronze is the source of truth.

### Time travel is your safety net

Delta Lake's automatic versioning saved May 12 data when a bug overwrote it. Run `DESCRIBE HISTORY table_name` to see versions, then `.option('versionAsOf', N)` to read any of them. Default retention: 30 days.

### Schema evolution must be explicit

Adding a column requires either `autoMerge=true` (session-level) or `overwriteSchema=true` (per-write). Production should use deliberate `ALTER TABLE`. During development, autoMerge is fine.

### Two SQL dialects coexist in Fabric

| Operation | T-SQL (SQL Endpoint) | Spark SQL (Notebooks) |
|---|---|---|
| Cast to date | `CAST(col AS DATE)` | `DATE(col)` |
| Top N rows | `SELECT TOP 10 ...` | `LIMIT 10` |
| String length | `LEN(col)` | `LENGTH(col)` |
| Current time | `GETDATE()` | `CURRENT_TIMESTAMP()` |

### Default semantic models are static

New Lakehouse tables don't auto-sync. Plan ahead: recreate the model when adding tables, or design table additions in advance.

### Pipelines need BOTH alerts AND heartbeats

- **Alerts on failure** — tell you when something is broken
- **Heartbeats on success** — confirm pipeline is healthy and running
- **Together** — you can distinguish "pipeline broke" from "pipeline silently stopped running"

### VADER's quirks (and why ML is hard)

VADER scored *"Supreme Court hands Alabama major boost in redistricting fight"* as +0.61 because of `hands`, `major`, `boost`. A human reader would interpret this depending on political context. **Lesson:** rule-based sentiment is a first approximation. Transformer models catch nuance better, but cost more compute.

---

## 6. Operations Guide

### Daily Health Check

1. Open News Dashboard v3
2. Check the line chart — today's date should be present
3. Scan inbox for any `⚠️ News Pipeline Failed` emails
4. Check Teams for the latest `✅ News Pipeline ran successfully` message

### Pipeline Run History

1. Workspace → `news_pipeline_orchestrator` → `⋯` menu → **View run history**
2. Look for ✅ Succeeded status on every step
3. Click any run for per-activity drill-in

### Manually Triggering a Run

1. Open `news_pipeline_orchestrator`
2. Top toolbar → ▶ **Run** → **Save and run**

### Modifying the Schedule

1. Open `news_pipeline_orchestrator`
2. Top toolbar → **Schedule**
3. Adjust frequency / timezone / end date → Save

### Recovery From Data Loss

```python
# 1. View history
spark.sql("DESCRIBE HISTORY top_headlines").show(20, False)

# 2. Read a specific version
df = spark.read.format("delta").option("versionAsOf", 4).table("top_headlines")

# 3. Validate, then merge back into live table
# (use the same MERGE pattern as fetch_news)
```

### Rotating API Keys

1. Log in at https://newsapi.org/account → reset/regenerate API key
2. Update Cell 1 of `fetch_news` (preferred: move to Azure Key Vault)
3. Save notebook

> ⚠️ **Trial capacity expiry** — Power BI trial expires ~July 4, 2026 (53 days from May 12). Before then: add Fabric capacity, turn off schedule, or migrate to paid workspace.

---

## 7. Next Steps

### Quality Hardening
- Data validation notebook between Bronze and Silver (row counts, null checks)
- Great Expectations for declarative tests
- Unit tests for `categorize()` function

### Security
- Move News API key to Azure Key Vault (`mssparkutils.credentials.getSecret(...)`)
- Workspace-level access controls
- Rotate any leaked credentials

### Scalability
- Parameterize country code in `fetch_news`
- ForEach loop in pipeline iterating over countries
- Use `/everything` endpoint for historical backfill

### Better ML
- Replace VADER with Hugging Face transformer (`distilbert-base-uncased-finetuned-sst-2-english`)
- Topic modeling (LDA via gensim, or BERTopic)
- Named Entity Recognition (NER) via spaCy

### Observability
- Data freshness card on dashboard ("Last update: 2 hours ago")
- Azure Application Insights connection
- Dedicated "Pipeline Health" dashboard page

### Certification Path

You're well-positioned for:
- **DP-700** — Implementing Data Engineering Solutions Using Microsoft Fabric
- **DP-600** — Implementing Analytics Solutions Using Microsoft Fabric

The exam covers exactly what was built. Your pipeline IS the prep material.

---

## 8. Glossary

| Term | Definition |
|---|---|
| **Bronze layer** | Raw data as ingested. Source of truth. |
| **Silver layer** | Cleaned, enriched, deduplicated data ready for analysis. |
| **Gold layer** | Business-ready aggregates (not built here — natural next step). |
| **Delta Lake** | Storage format adding ACID transactions, schema evolution, time travel to data lakes. |
| **DirectLake** | Fabric's storage mode for semantic models — queries Lakehouse directly. |
| **Lakehouse** | Fabric storage combining data lake (files) + warehouse (SQL) features. |
| **MERGE** | SQL upsert — insert OR update based on match condition. |
| **Idempotent** | Property where re-running produces same result as running once. Critical for scheduled pipelines. |
| **Medallion architecture** | Layered design: Bronze → Silver → Gold. |
| **Semantic model** | Power BI's data layer — wraps tables, defines relationships and measures. |
| **UDF** | User Defined Function — Python wrapped for Spark row-by-row execution. |
| **VADER** | Valence Aware Dictionary and sEntiment Reasoner — rule-based sentiment library for short text. |
| **Window function** | SQL op computing values across related rows (`row_number()`, `rank()`). |
| **T-SQL** | Microsoft SQL Server's SQL dialect (Fabric SQL analytics endpoint). |
| **Spark SQL** | Apache Spark's SQL dialect (notebooks via `spark.sql()` and DataFrame API). |

---

*This pipeline runs unattended every 6 hours. Failures send email. Successes ping Teams. The dashboard updates within seconds via DirectLake. Welcome to production data engineering.*
