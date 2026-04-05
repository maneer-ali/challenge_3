# Challenge 3 — System Design Document

# AIRA Scalable Recommendation Platform (GCP-Native Architecture)

**Author:** Your Name
**Date:** April 2026

---

# 1. Overview

AIRA has grown significantly and now supports:

* 500 tenants
* 50M events/day
* 10M customers
* 5,000 recommendation API requests/second at peak
* 2M direct mail pieces/month requiring personalized URLs (PURLs) and USPS tracking

The existing architecture must be redesigned so that event ingestion, feature generation, recommendation serving and tenant isolation scale without increasing latency or operational complexity.

This design uses managed Google Cloud services to minimize infrastructure management while enabling near-real-time recommendations.

---

# 2. Capacity Assumptions

## Event Throughput

50M events/day:

50,000,000 / 86,400 = **~580 events/sec average**

Assuming peak traffic is 5x average:

**~3,000 events/sec peak**

## Event Storage Estimate

Assume average event size = 1 KB

Daily raw storage:

50 GB/day

Annual raw storage:

18 TB/year raw

Using Parquet compression (~5x):

~3.5 TB/year effective storage

## Recommendation API Load

5,000 req/sec peak requires:

* sub-100 ms feature retrieval
* cache-first recommendation serving
* fallback when ML endpoints are unavailable

---

# 3. A. Event Pipeline at Scale

## Problem

Direct writes into relational databases are no longer efficient because:

* write throughput creates lock contention
* analytical workloads impact transactional performance
* storage grows rapidly

## Proposed Architecture

```text
Applications / SDKs
        ↓
Pub/Sub
        ↓
Dataflow
        ↓
Cloud Storage (Parquet)
        ↓
BigQuery
```

---

## Event Ingestion Layer

Use Google Cloud Pub/Sub for event ingestion.

### Why Pub/Sub

* fully managed messaging system
* automatic horizontal scaling
* durable event retention
* no broker management required

## Throughput Design

Peak:

3,000 events/sec

Pub/Sub supports this without manual partition provisioning.

## Ordering Strategy

Ordering key:

```text
tenant_id + customer_id
```

This preserves:

* customer event ordering
* tenant locality

---

## Stream Processing Layer

Use Google Cloud Dataflow.

### Dataflow Responsibilities

* event deduplication
* enrichment
* rolling aggregations
* session windows
* streaming feature extraction

## Exactly-Once Semantics

Dataflow provides:

* checkpointing
* replay-safe processing
* watermark-based late event handling

This ensures consistent feature computation.

---

## Data Lake Storage

Store processed events in Cloud Storage using:

* Parquet format

Partition strategy:

```text
/date=YYYY-MM-DD/tenant_id=123/
```

Avoid customer-level partitioning to prevent tiny files.

---

## Analytical Store

Load curated event data into BigQuery.

### BigQuery Stores

* historical events
* tenant-level analytics
* model training datasets
* campaign performance data

---

## PostgreSQL / Cloud SQL Stores Only

* tenant configuration
* campaign metadata
* active customer metadata
* recommendation metadata

Operational data remains separate from analytical workloads.

---

# 4. B. Feature Store Design

## Problem

The current batch feature pipeline takes 4 hours and blocks model training.

## Goal

Near-real-time feature generation supporting:

* batch features
* streaming features

## Proposed Architecture

```text
Batch Features + Streaming Features
        ↓
Unified Feature Store
        ↓
Recommendation API
```

---

## Batch Feature Pipeline

Use Dataflow batch jobs orchestrated by Cloud Composer.

### Batch Features

Examples:

* 30-day spend
* category affinity
* churn probability
* average purchase value

Batch refresh frequency:

daily

---

## Streaming Feature Pipeline

Dataflow updates streaming features continuously.

### Streaming Features

Examples:

* clicks in last 5 minutes
* cart activity
* recent category interactions
* session engagement score

---

## Online Feature Store

Use Memorystore (Redis).

### Key Design

```text
tenant_id:user_id
```

### Why Memorystore

* 1–3 ms latency
* ideal for online serving
* supports sub-100 ms recommendation API target

---

## Offline Feature Store

Use BigQuery.

This supports:

* reproducible model training
* historical backfills
* feature versioning

---

## API Read Pattern

Recommendation API reads:

1. latest online features from Memorystore
2. fallback batch features if streaming features unavailable

Target latency:

<20 ms feature retrieval

---

# 5. C. Recommendation Serving Architecture

## Problem

At 5,000 req/sec, full real-time inference for every request becomes expensive.

## Hybrid Strategy

Use:

* precomputed recommendations
* lightweight real-time reranking

---

## Precomputation Layer

Generate top 100 products per user periodically.

Frequency:

every 1–2 hours

Store results in Memorystore.

---

## Real-Time Serving Flow

```text
API Request
   ↓
Memorystore Candidate Fetch
   ↓
Feature Fetch
   ↓
Model Scoring
   ↓
Response
```

---

## Model Hosting

Use Vertex AI endpoint.

### Vertex AI Handles

* model deployment
* autoscaling
* version management
* inference serving

---

## Cache Invalidation

When product catalog changes:

publish product_update event to Pub/Sub

Consumers invalidate affected cache keys.

---

## A/B Testing Design

Stable assignment using:

```text
hash(user_id) % 100
```

### Example

0–49 → model A
50–99 → model B

This ensures the same user remains in the same experiment group.

---

## Graceful Degradation

If Vertex AI is unavailable:

1. serve cached recommendations
2. tenant-level best sellers
3. global popular products

This avoids API outage.

---

## API Infrastructure

Deploy recommendation API on GKE.

### Why GKE

* autoscaling
* rolling deployment
* high concurrency support

---

# 6. D. Multi-Tenant Data Isolation at Scale

## Problem

500 tenants have very different usage patterns.

Large tenants can affect smaller tenants.

---

## Shared Schema vs Dedicated Schema

## Shared Schema

### Pros

* simpler maintenance
* easier migrations

### Cons

* noisy neighbor risk

---

## Tenant Per Schema

### Pros

* stronger isolation

### Cons

* higher operational overhead

---

## Recommended Hybrid Strategy

## Small Tenants

Shared schema

## Large Tenants

Dedicated Cloud SQL instance

Threshold:

tenants contributing >5% of total platform load

---

## Relational Database Choice

Use Cloud SQL for PostgreSQL.

### Why Cloud SQL

* managed backups
* failover
* read replicas

---

## Connection Pooling

Use PgBouncer.

## RLS Constraint

RLS with transaction pooling creates session challenges.

## Solution

Prefer application-enforced tenant filtering.

---

## Noisy Neighbor Mitigation

Apply:

* tenant-level API quotas
* query timeout
* background workload controls

---

# 7. Monitoring and Operations

## Monitoring Stack

Use:

* Cloud Monitoring
* Cloud Logging
* Grafana

---

## Key Metrics

* Pub/Sub backlog
* Dataflow lag
* Redis hit ratio
* API latency
* feature freshness

---

## Failure Handling

## Pub/Sub Backlog

Scale Dataflow workers

## Memorystore Failure

Fallback to batch snapshot

## Vertex AI Failure

Serve cached recommendations

---

# 8. Migration Strategy

## Phase 1

Dual-write legacy pipeline + Pub/Sub

## Phase 2

Validate data parity

## Phase 3

Migrate tenants gradually

## Phase 4

Retire legacy pipeline

---

# 9. Direct Mail Personalization

## PURL Generation

Generate unique links:

```text
tenant-domain.com/p/{hashed_customer_id}
```

Store mappings in Cloud SQL.

---

## USPS Tracking

Tracking events enter Pub/Sub pipeline.

This allows mail attribution in recommendation models.

---

# 10. Final Architecture Summary

```text
Applications
   ↓
Pub/Sub
   ↓
Dataflow
   ↓
Cloud Storage
   ↓
BigQuery
   ↓
Memorystore
   ↓
Vertex AI + GKE API
```

---

# 11. Architectural Rationale

Because the system is event-heavy and analytics-driven, managed GCP services reduce operational burden while supporting near-real-time feature generation.

This allows engineering effort to focus on product evolution rather than infrastructure maintenance.