# Real-Time YouTube Analytics (Data Stream)

[![Portfolio](https://img.shields.io/badge/my_portfolio-000?style=for-the-badge&logo=ko-fi&logoColor=white)](https://ridwanbankole.github.io/)
[![LinkedIn](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/bankoleridwan/)

![Banner](https://raw.githubusercontent.com/RidwanBankole/Real-Time-YouTube-Analytics/refs/heads/main/img/youtube%20analytics.jpg)


## The Problem

Content teams on YouTube are flying blind. They publish a video, check the numbers a few
days later, and by the time a pattern becomes visible — a content format that's gaining
traction, a topic that's losing audience interest, a posting time that consistently
underperforms — the opportunity to act on it has already passed.

The problem isn't a lack of data. YouTube generates enormous amounts of it: views, watch
time, engagement rates, subscriber movement, comment sentiment, thumbnail performance. The
problem is that most teams access it through a static dashboard that shows what happened
last week — not what is happening right now.

A content strategy built on delayed data is always reacting. The goal here was to build a
system that puts teams ahead of the curve instead.


## What I Built

A real-time YouTube analytics platform that continuously extracts video and channel metrics
from the YouTube Data API, streams them through a live data pipeline, and surfaces them in
an always-current analytics dashboard — giving content teams the intelligence to make
decisions based on what is happening, not what happened.

![Architecture Diagram](IMAGE_ARCHITECTURE_PLACEHOLDER)

The platform tracks:

| Metric Category | Metrics |
|---|---|
| **Engagement** | View counts, like/dislike ratios, comment volume, engagement rate trends |
| **Audience** | Subscriber growth, watch time, audience retention |
| **Content** | Video duration, tags, category classification, caption availability |
| **Visual** | Thumbnail quality scores |

All data flows in real time — from API extraction through streaming ingestion, processing,
and into the warehouse — with a live dashboard consuming the latest state of every metric
continuously.


## Architecture & My Contribution

**Data Extraction — YouTube Data API v3**

The pipeline begins with a Python extraction layer querying the YouTube Data API v3. I
engineered the extraction to capture the full breadth of available signals — not just the
obvious view counts and likes, but thumbnail quality scoring, caption availability flags,
and category classification that give content strategy analysis real depth. Getting
consistent, well-structured data out of an API that returns nested, inconsistently typed
JSON responses required deliberate schema design from the start.

**Real-Time Streaming — Apache Kafka**

Extracted events are published to Apache Kafka topics for real-time ingestion. Kafka
decouples the extraction layer from the processing layer — meaning the pipeline continues
ingesting data even when downstream processing is under load, and no events are lost during
peak periods. I designed the topic structure and partitioning strategy to support parallel
processing across multiple channels simultaneously.

![Kafka Pipeline](IMAGE_KAFKA_PLACEHOLDER)

**Stream Processing — Spark Streaming**

Incoming Kafka events are consumed and processed by Spark Streaming, which handles
transformation, enrichment, and aggregation in flight — before the data ever reaches the
warehouse. This is where engagement rate calculations, time-window aggregations, and
trend detection happen in real time, rather than being computed retrospectively in SQL
after the fact.

**Cloud Data Warehouse — BigQuery**

Processed events land in BigQuery as the cloud data warehouse. I designed the schema for
analytical query performance — partitioned by date and clustered by channel and content
category to ensure dashboard queries run fast even as the dataset grows to hundreds of
millions of rows.

![BigQuery Schema](IMAGE_BIGQUERY_PLACEHOLDER)

**Orchestration — Apache Airflow**

The full pipeline is orchestrated with Apache Airflow, which handles scheduling, dependency
management, and error alerting. Any pipeline failure — API rate limiting, Kafka lag, Spark
job failure — triggers an immediate alert, keeping the platform reliable without manual
monitoring.

**Compliance — PII Masking & Encryption**

From day one, I enforced data masking on all PII fields in the pipeline and applied
encryption both at rest and in transit. For a platform processing user-generated content
and comment data at scale, compliance isn't an afterthought — it's a structural requirement
built into the pipeline architecture itself.


## The Outcome

![Dashboard](IMAGE_DASHBOARD_PLACEHOLDER)

The platform delivers a live analytics dashboard that tracks channel performance, content
strategy trends, and audience behaviour at scale — updating continuously as new data flows
through the pipeline. Content teams can now see engagement trends forming in real time,
identify high-performing content formats as they emerge, and act on audience behaviour
signals while they are still relevant.


## Tech Stack

| Layer | Tools |
|---|---|
| Data Extraction | Python, YouTube Data API v3 |
| Streaming Ingestion | Apache Kafka |
| Stream Processing | Apache Spark Streaming |
| Cloud Data Warehouse | Google BigQuery |
| Pipeline Orchestration | Apache Airflow |
| Compliance | PII Masking, Encryption at Rest & in Transit |


## What I'd Do Differently

- **Sentiment analysis on comments** — comment volume is a useful engagement signal, but
  comment *sentiment* is far more informative. Adding an NLP layer to classify comment
  tone in the stream processing stage would surface whether audience reaction to a video
  is positive, critical, or neutral — a qualitative signal that view counts alone cannot
  provide.
- **Predictive performance scoring** — the current platform is descriptive: it tells you
  what is happening. A next layer would be predictive: using historical performance patterns
  to forecast how a newly published video is likely to perform within its first 24 hours,
  giving teams an early signal before the algorithm amplifies or suppresses it.
- **Competitor benchmarking** — pulling metrics from competitor channels through the same
  pipeline would allow content teams to contextualise their own performance against the
  market, not just against their own historical baseline.
- **Alerting on anomalies** — beyond pipeline health alerts in Airflow, building a metric
  anomaly detection layer that flags unusual spikes or drops in engagement in real time
  would allow teams to respond to viral moments or sudden drops in reach immediately.


## Let's Talk

If you're working on a data engineering, real-time analytics, or cloud data platform
problem and need someone who can architect and deliver the full pipeline —
[get in touch](https://ridwanbankole.github.io/#contact).
