# Real-Time YouTube Analytics Platform

[![Portfolio](https://img.shields.io/badge/my_portfolio-000?style=for-the-badge&logo=ko-fi&logoColor=white)](https://ridwanbankole.github.io/)
[![LinkedIn](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/bankoleridwan/)

![Banner](https://raw.githubusercontent.com/RidwanBankole/Real-Time-YouTube-Analytics/refs/heads/main/img/youtube%20analytics%202.jpg)


## The Problem

Content teams on YouTube are making strategy decisions on week-old data. They publish a
video, check the numbers days later, and by the time a pattern becomes clear — a format
gaining traction, a topic losing audience interest, a posting window that consistently
underperforms — the opportunity to act on it has already closed.

The data isn't missing. YouTube generates enormous amounts of it. The problem is that
most teams access it through a static report that shows what happened last week, not what
is happening right now. A content strategy built on delayed data is always reacting. The
goal here was to build a system that puts teams ahead of the curve instead.


## What I Built

A real-time YouTube analytics platform that continuously extracts video and channel metrics
from the YouTube Data API v3, streams them through a live data pipeline, and surfaces them
in an always-current dashboard — so content teams can make decisions based on what is
happening now, not what happened last week.

### Pipeline Architecture

```
┌─────────────────────┐
│  YouTube Data API v3 │  ← Python extraction layer
│  (video + channel    │    view counts, likes, comments,
│   metrics, PII       │    engagement rate, duration, tags,
│   masked at source)  │    category, caption, thumbnail score,
└──────────┬──────────┘    subscriber count
           │
           ▼
┌─────────────────────┐
│    Apache Kafka      │  ← Real-time streaming ingestion
│  Topic: yt_metrics   │    Decouples extraction from processing
│  3 partitions        │    No data loss under downstream load
│  Snappy compression  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐       ┌──────────────────────────┐
│  Spark Structured    │       │     Apache Airflow        │
│  Streaming           │       │  (Orchestration layer)    │
│                      │       │  - Schedules pipeline     │
│  - Parse JSON events │       │  - Monitors health        │
│  - Compute derived   │       │  - Data quality checks    │
│    fields (engagement│       │  - Email alerts on        │
│    rate, duration    │       │    failure                │
│    mins, publish     │       └──────────────────────────┘
│    hour/day/month)   │
│  - Aggregate channel │
│    rollups           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│     BigQuery         │  ← Cloud data warehouse
│                      │    Partitioned by day
│  video_metrics       │    Clustered by channel + category
│  channel_aggregates  │    Controlled dashboard_view (PII excluded)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Streamlit Dashboard │  ← Live analytics UI
│  (refreshes every    │    Engagement trends, publish patterns,
│   5 minutes)         │    content strategy signals
└─────────────────────┘
```


## Metrics Captured

| Category | Metrics |
|---|---|
| **Engagement** | View counts, like counts, comment volume, engagement rate |
| **Audience** | Subscriber count per channel |
| **Content** | Video duration, tags, category classification, caption availability |
| **Visual** | Thumbnail quality score (resolution-based, 1–5) |
| **Derived** | Engagement rate, duration in minutes, is\_long\_form flag, publish hour, publish day of week, publish month |


## How I Built It — Layer by Layer

### Data Extraction — YouTube Data API v3

The pipeline starts with a Python extraction layer that queries the YouTube Data API v3
for every video in a channel's upload history. I engineered the extraction to go beyond
the obvious metrics — view and like counts — and capture thumbnail quality scores, caption
availability, and category classification, which are the signals that give content strategy
analysis real analytical depth.

The API returns nested, inconsistently typed JSON. Getting clean, consistently structured
output required explicit schema design and type coercion from the start — any ambiguity
here compounds into data quality problems downstream.

All free-text fields (video descriptions, titles) pass through a PII masking layer before
leaving the extractor, stripping email addresses and phone numbers before they enter the
pipeline.


### Real-Time Streaming — Apache Kafka

![Kafka Flow](https://raw.githubusercontent.com/RidwanBankole/Real-Time-YouTube-Analytics/refs/heads/main/img/Kafka%20flow.drawio.png)

Extracted events are published to a Kafka topic (`youtube_metrics`) with three partitions,
keyed on `video_id` to ensure ordered processing per video. Snappy compression reduces
network and disk overhead on high-volume runs.

The critical design decision here was **decoupling extraction from processing**. Without
Kafka in the middle, a slow Spark job would back up the entire pipeline. With Kafka, the
extractor publishes at its own pace and Spark consumes at its own pace — independently,
without either blocking the other. No events are lost if downstream processing is
temporarily under load.


### Stream Processing — Apache Spark Structured Streaming

Spark consumes from Kafka and handles all transformation and enrichment **in flight**,
before any data reaches BigQuery. This is where engagement rate is calculated, duration is
converted from ISO 8601 to minutes, `is_long_form` is derived, and publish timestamps are
decomposed into hour, day-of-week, and month fields for time-pattern analysis.

Computing these in Spark rather than retrospectively in SQL means the warehouse always
contains analysis-ready data — no post-load transformation jobs required.

Two sinks run in parallel:
- **video_metrics** — one row per video, appended every 30 seconds
- **channel_aggregates** — channel-level rollups, updated every 60 seconds


### Cloud Data Warehouse — BigQuery

Processed events land in BigQuery, designed for analytical query performance from day one:

- **Partitioned by day** on `processing_time` — queries that filter by date scan only
  the relevant partitions, not the full table
- **Clustered by `channel_id` and `category_id`** — dashboard queries that filter by
  channel or content category run significantly faster at scale
- **Controlled `dashboard_view`** — a BigQuery view that deduplicates rows (keeping
  only the most recent extract per `video_id`) and excludes the description field,
  which carries PII risk in free text

The dashboard queries this view, never the raw table directly.


### Orchestration — Apache Airflow

The full pipeline runs on a 6-hour schedule managed by Airflow. Each DAG run:

1. Extracts from YouTube API and publishes to Kafka
2. Runs data quality checks against BigQuery (row count, null `video_id` check)
3. Branches — if checks pass, refreshes channel aggregates; if they fail, fires an
   alert email and halts
4. Logs a completion summary with events published and channels processed

Any infrastructure failure — API rate limiting, Kafka lag, Spark job error — triggers an
immediate email alert. The pipeline never fails silently.


### Compliance — PII Masking & Encryption

| Control | Where Applied |
|---|---|
| PII masking (email, phone) | Extractor — before Kafka |
| Encryption in transit | All API and BigQuery connections (TLS) |
| Encryption at rest | BigQuery default (Google-managed keys) |
| PII field exclusion | `dashboard_view` excludes description column |
| Deduplication | `dashboard_view` surfaces only latest extract per video |

Compliance was designed into the pipeline architecture, not bolted on afterwards.


## The Outcome

Content teams get a live dashboard that updates every five minutes — tracking channel
performance, engagement trends, and audience behaviour as they develop, not after the fact.

The dashboard surfaces:
- Which videos are gaining or losing engagement in real time
- The publish hours and days that consistently produce higher engagement
- Whether long-form or short-form content performs better for a given channel
- The correlation between thumbnail quality score and view count
- Channel-level subscriber and view trends over any selected time window


## Tech Stack

| Layer | Tools |
|---|---|
| Language | Python 3.11 |
| Data Extraction | YouTube Data API v3 |
| Streaming Ingestion | Apache Kafka 3.6 (Confluent) |
| Stream Processing | Apache Spark Structured Streaming 3.5 |
| Cloud Data Warehouse | Google BigQuery |
| Pipeline Orchestration | Apache Airflow 2.9 |
| Dashboard | Streamlit + Plotly |
| Containerisation | Docker, Docker Compose |
| Compliance | PII masking, TLS, BigQuery encryption at rest |


## What I'd Do Differently

**Comment sentiment analysis** — comment volume tells you how much conversation a video
sparked. Comment *sentiment* tells you whether that conversation was positive, critical,
or polarised — a qualitative signal that view counts alone can't provide. Adding an NLP
layer in the Spark processing stage to classify comment tone in-flight would make the
engagement picture significantly more complete.

**Predictive performance scoring** — the current platform is descriptive: it tells you
what is happening. The natural next layer is predictive: using a video's first-hour
metrics as input features to forecast its 7-day performance, giving content teams an
early signal before the algorithm has fully decided whether to amplify or suppress it.

**Competitor benchmarking** — pulling the same metrics from competitor channels through
the same pipeline would let content teams contextualise their own performance against the
market rather than just their own historical baseline. The pipeline already supports
multiple channel IDs; the analysis layer is the gap.

**Anomaly detection alerting** — Airflow currently alerts on pipeline failures.
A separate layer that flags unusual spikes or drops in *metric values* — a video
suddenly gaining 10x normal engagement, or a channel's subscriber count dropping
overnight — would let teams respond to both opportunities and problems in real time.


## Let's Talk

If you're working on a data engineering, real-time analytics, or cloud data platform
problem — [get in touch](https://ridwanbankole.github.io/#contact).
