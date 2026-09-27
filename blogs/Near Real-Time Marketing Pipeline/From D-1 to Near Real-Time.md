---
title: From D-1 to Near Real-Time
slug: from-d-1-to-near-real-time
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - starrocks
  - real-time
  - architecture
description: How we replaced a D-1 batch pipeline with a windowed near real-time marketing data platform, and why ingestion and processing had to be separated.
reading_time: 12
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 1
---

This series walks through a near real time marketing pipeline we built: how it replaced a daily batch setup, and the design calls behind ingestion, windowing, state, and delivery. This first post covers the problem and the overall shape of the system. Later posts will go deeper into each stage.

## The problem: D-1 marketing data

Our original marketing data pipeline was a daily batch setup. Spark jobs ran overnight on transient cloud compute, orchestrated through a workflow scheduler, and the resulting datasets were exported to downstream marketing platforms.

That worked fine for daily reporting. The constraint was simple: marketing could only react to yesterday's data. When user behavior needs to change what happens today, waiting until tomorrow is a long time.

## The new goal

We moved from a daily batch model to a windowed near real time setup:

```text
Day N data
    ↓
Overnight processing
        ↓
Day N+1 activation
```

became:

```text
Continuous ingestion
        ↓
Short processing window
        ↓
Classification
        ↓
Enrichment
        ↓
Delivery
```

The platform handles roughly 90 to 100 million raw transaction records per day, with about 60 to 70 million left after deduplication. Production classifies on an hourly cadence, and the orchestrator supports configurable window sizes, including 30 minute windows.

## Why we needed a different processing engine

The old stack was built around batch Spark processing. During evaluation we recorded a concrete pain point: a cold query over the hot transaction dataset could take more than 20 minutes.

So the new design put StarRocks in as the analytical layer. Classification queries now run over freshly arriving transaction data at far lower latency.

## High level architecture

The system looks like this:

```text
                Source Kafka
                    │
                    ▼
              Routine Load
                    │
                    ▼
              Raw Data Table
                    │
                    ▼
        ┌─────────────────────────┐
        │   Pipeline Orchestrator │
        │                         │
        │  1. Config Sync         │
        │  2. Deduplication       │
        │  3. Classification      │
        │  4. State Snapshot      │
        │  5. State Update        │
        │  6. Publish             │
        └────────────┬────────────┘
                     │
                     ▼
              Classified Events
                     │
                     ▼
             Enrichment Service
                     │
                     ▼
        Vendor-Specific Kafka Topics
                     │
                     ▼
             Delivery Workers
                     │
             ┌───────┴────────┐
             ▼                ▼
          Success          Failed Events
                                │
                                ▼
                               DLQ
```

Kafka ingestion runs continuously through Routine Load. Then comes windowed orchestration, classification with state handling, Kafka publishing, enrichment, vendor specific routing, and delivery.

## Why windowed processing

The core decision was to split continuous ingestion from windowed processing. Kafka keeps receiving events without pause, while the orchestrator works on a closed classification window:

```text
Kafka → StarRocks
```

```text
Closed window
      ↓
Deduplicate
      ↓
Classify
      ↓
Update state
      ↓
Publish
```

The orchestrator fires often but only touches a window after it has closed and the settlement period has passed. Ingestion never waits, and processing never races half arrived data.

## What happens inside one window

Each run follows the same sequence:

```text
1. Synchronize configuration
2. Deduplicate transactions
3. Classify users
4. Snapshot existing state
5. Update current state
6. Publish classified event deltas
```

Classification here is stateful. The question is not which users showed up in this window. It is what each user's latest classification looks like, given the new transactions plus whatever state already exists:

> "Which users appeared in this window?"

That alone is not enough. We need the user's latest classification state from the new transactions and the stored state.

## Configuration driven classification

Classification rules live in a relational configuration database, not inside the application code. At the start of each run, the orchestrator pulls a copy of the rules and syncs them into the analytical layer. The Go based orchestrator then builds the classification queries from those rules.

Since rules are snapshotted for the run, changing a rule mid run cannot alter that run:

> **Changing a classification rule does not require redeploying the pipeline application.**

Operations gets a useful property out of this: a rule edit takes effect on the next run, never the current one.

## Deduplication

One user can produce many transactions, and duplicates have to go before classification. The dedup key is:

```text
user_id + transaction_id
```

The pipeline builds a deduplicated view of the raw transactions first, then classifies. At our scale that means roughly 90 to 100M raw records a day coming down to about 60 to 70M deduplicated records.

## Stateful classification

Every user carries a current classification state. Before touching it, the pipeline snapshots it:

```text
Current State
     │
     ▼
State Snapshot
     │
     ▼
Backup State
```

Then the new results go in:

```text
Temporary Classification
          │
          ▼
      State Update
          │
          ▼
     Current State
```

If a later stage fails, there is something to come back to.

## Recovery from state corruption

The backup is a recovery point, not an audit trail. When a classification run fails after changing state, the previous state gets restored from backup before the window is retried. State updates run as a controlled sequence:

```text
Snapshot old state
       ↓
Classify
       ↓
Update state
       ↓
Publish
```

Snapshot, classify, update, and publish stay separate orchestrator stages so a retry starts from known ground.

## Publishing with at least once semantics

After classification, the pipeline publishes the event delta to Kafka in chunks and persists its progress. That leaves one failure case open:

```text
Kafka write succeeds
        ↓
Process crashes
        ↓
Progress checkpoint was not written
        ↓
Chunk is published again
```

So the delivery boundary is at least once. Chunked publishing, persisted progress, replay after a crash, and UUID based event identity for repeats are all part of the design.

*That covers the shape of the system. Next up: how the window boundaries and settlement period actually work, and what we learned tuning them in production. Till then enjoy your life and happy engineering!*
