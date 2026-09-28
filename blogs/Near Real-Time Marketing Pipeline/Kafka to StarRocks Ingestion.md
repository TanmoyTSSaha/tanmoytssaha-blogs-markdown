---
title: Kafka to StarRocks Ingestion
slug: kafka-to-starrocks-ingestion
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - starrocks
  - ingestion
  - real-time
description: How transaction events flow from Kafka into StarRocks through Routine Load, and why ingestion runs on a different clock than classification.
reading_time: 10
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 2
---
Last time I covered the overall shape of the pipeline. This post goes into the ingestion layer: how transaction events get from Kafka into StarRocks, and why that path runs on its own clock while classification works in closed windows.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=2uU7HtVk0m7-BiY0m79j">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=2uU7HtVk0m7-BiY0m79j&type=embed" /></a>

## The path

```text
Source Kafka Topic
        |
        | Avro events
        | Schema Registry
        v
StarRocks Routine Load
        |
        | business-valid transaction filter
        v
Raw Transaction Table
        |
        v
Pipeline Orchestrator
        |
        | reads a closed [window_start, window_end) slice
        v
Deduplicated Transaction Table
```

Routine Load is the ingestion writer. Deduplication comes later as a SQL stage run by the orchestrator, which never touches Kafka directly.

## Why Kafka is the ingestion boundary

Upstream transaction data already arrives as Avro events on Kafka, so the ingestion layer treats the topic as its upstream contract instead of reading from a source database.

StarRocks consumes Kafka natively through Routine Load. That keeps the ingestion path short, with business validity filtering applied at the storage boundary. The consumer is a StarRocks Routine Load job, not a custom Go or Python service. Schema Registry deserializes the Avro records, and the load mapping shapes each event into the raw table schema.

A new ingestion job starts from a configured Kafka offset rather than replaying all of history. Broker and Schema Registry connectivity come from deployment configuration.

## Why StarRocks is the analytical layer

Classification has to scan recent transaction history and join it against a large per user state table, over and over on a schedule. That asks several things of one engine:

- SQL over a multi-day hot transaction dataset with low query latency
- Upsert-style state management for current user state
- Native Kafka ingestion with filtering
- A MySQL-compatible connection interface for the Go services
- Efficient columnar scans and analytical joins

We compared those needs against other engines. A cold batch query over the hot transaction set took more than 20 minutes on the old Spark and Hive path, while the new design needed interactive analytical queries over the same data in seconds. StarRocks covered ingestion, analytical queries, and state handling together, which is why it won.

Ingestion runs at about 90 to 100 million raw transaction records a day, with roughly 60 to 70 million left after deduplication. Hot data stays around for a short recent history window, and older data gets archived to object storage.

## Routine Load

Routine Load runs as a continuous job, and StarRocks commits each load batch atomically. Its setup looks like this:

| Property | Public description | Role |
|---|---|---|
| Format | Avro | Deserialize events through Schema Registry |
| Batch size | Large bounded batch | Prevent excessive commit overhead while keeping ingestion flowing |
| Commit interval | Short periodic interval | Keep data continuously available to downstream processing |
| Parallelism | Multiple load tasks | Increase ingestion throughput |
| Error threshold | Configured error budget | Pause ingestion when malformed data exceeds the accepted threshold |
| Time zone | IST | Keep ingestion timestamps aligned with orchestration windows |

The ingestion filter keeps only business valid transaction records. It deliberately does not use optional identity fields as a gate, because real payments can arrive without some secondary identifiers. Identity gets sorted out later in deduplication.

Columns are mapped explicitly during Routine Load instead of relying on table defaults. Optional source fields get null-safe mappings with controlled fallbacks where needed, and epoch timestamps convert into the platform's operational time zone on the way in.

One distinction matters everywhere downstream:

```text
Transaction time  = when the payment occurred
Ingestion time    = when the platform stored the event
```

A payment that happened earlier but arrived late keeps its original transaction timestamp and picks up a later ingestion timestamp.

Frequently queried context fields are extracted into typed columns. The full nested context payload is also kept as JSON, so new fields become usable later without touching the ingestion contract.

## Raw table design

The raw transaction table is append oriented. It uses a duplicate key model around customer and transaction identity, and that is not a uniqueness constraint. A Kafka redelivery can write a second physical row, which is expected at this layer. Deduplication happens later on purpose:

```text
Kafka delivery
     |
     v
Append raw row
     |
     v
Windowed deduplication
     |
     v
Canonical transaction set
```

The table is partitioned by ingestion time and distributed by customer identity, so later customer centric joins stay cheap. Raw and deduplicated tables share compatible partition and distribution strategies, which keeps the classification workload on a predictable layout. Frequently filtered attributes get a small amount of indexing, and compression keeps storage down.

## Processing windows

Ingestion and classification run on different clocks. Production classifies on hourly windows. The orchestrator accepts shorter configurable intervals, including 30 minute windows, but the live setting is hourly.

The orchestrator gets polled on a period. A poll does not mean a window gets processed right away. A window runs only after it has closed and a short settlement period has passed to absorb ingestion lag.

Windows are defined on ingestion time:

```sql
WHERE ingested_at >= '${WINDOW_START}'
  AND ingested_at <  '${WINDOW_END}'
```

The half open interval keeps adjacent windows from overlapping. On an hourly schedule:

```text
09:00 ─────────────────── 10:00
        processing window

10:00 ── settlement ──> 10:05
                         |
                         +-- eligible for processing
```

The schedule polls. The window is the actual unit of work. Those are two separate things.

## Continuous ingest, discrete windows

Routine Load keeps ingesting while a classification window runs:

```text
Kafka
  |
  | continuous
  v
StarRocks Raw Table
  |
  | closed windows
  v
Pipeline Orchestrator
```

The orchestrator never reads Kafka. It reads rows already landed in StarRocks that fall inside a finished time range. Between the two clocks sits the settlement gap:

1. Kafka keeps receiving events.
2. Routine Load keeps appending rows.
3. The orchestrator checks periodically whether the next window closed.
4. Past the settlement period, that window gets processed.
5. Records arriving after close carry a later ingestion timestamp, so a later window picks them up.

Nobody syncs Kafka offsets with the classification engine. Kafka ingests on its own while StarRocks hands each window a stable slice to query.

## Handling late arrivals and duplicates

Because windows use ingestion time, a payment lands in the window where it first becomes visible to the analytical layer, even when the transaction itself is older. Deduplication has to cover both:

- duplicates that arrive within the same processing window
- duplicate or related records that arrive in a later window

The system collapses duplicates with the transaction identity plus supporting payment identifiers across a bounded history range. Inside one window, the first arrival wins a place in the deduplicated result. Stronger identifiers take precedence through an identity hierarchy, while records missing optional identity fields survive instead of getting dropped by mistake.

That splits the responsibilities cleanly:

```text
Continuous Kafka ingestion
          |
          v
Append-only raw storage
          |
          v
Windowed + historical deduplication
          |
          v
Stable input for classification
```

## The point of the split

Continuous ingestion keeps the analytical store current, and discrete windows hand classification a deterministic unit of work. With those two concerns apart, deduplication, state handling, rule evaluation, publishing, and recovery each become straightforward to reason about.

*Next up: the window lifecycle in detail, settlement tuning, and what late data taught us. Till then enjoy your life and happy engineering!*
