---
title: Deduplicating Millions of Records Efficiently in StarRocks
slug: deduplicating-millions-records-starrocks
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - starrocks
  - sql
  - deduplication
  - real-time
description: How the pipeline collapses duplicate payments with a three-tier identity model, inside one window and across windows.
reading_time: 12
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 4
---
Deduplication is its own SQL stage in the classification pipeline. It reads the current ingestion window from the raw transaction table and writes the survivors into a separate deduplicated table. Classification only ever reads the deduplicated side.
The core identity is not one key. The design runs a three-tier identity model, with `customer_id + transaction_id` as the last fallback.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=jTpC0tTJC2EM0vrpaynN">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=jTpC0tTJC2EM0vrpaynN&type=embed" /></a>

## The three-tier identity model

Every tier stays scoped to one customer. Inside that scope, three identity keys apply in order:

| Order | Identity | What it handles |
|---|---|---|
| 1 | `customer_id` + non-empty payment-group identifier | Multiple representations of the same payment that share a common payment/order identity. |
| 2 | `customer_id` + payment reference | Cases where the higher-level payment-group identifier differs but the payment reference is shared. |
| 3 | `customer_id` + `transaction_id` | Payments that do not carry the higher-level identifiers and exact redelivery of the same transaction. |

Tier 1 goes first because one payment can carry more than one payment reference. A reference-first strategy could split two representations of a single payment into two transactions, so the higher payment identity wins.

A missing payment-group identifier never counts as a shared value. A row without one falls back to its own transaction identity instead of matching strangers.

## Why duplicate rows exist

Duplicate physical rows come from several independent causes.

### Multiple representations of one payment

The upstream stream can carry more than one representation of the same payment. Those records may use different transaction identifiers while sharing a higher payment identity or payment reference.

### Kafka redelivery

Kafka delivers at least once. The same transaction can show up twice, inside one ingestion window or a later one.

### Retries or split representations

A retry or a related representation can keep the higher payment identity while carrying a different payment reference.

Ingestion makes no attempt to remove all of these. The raw table stays tuned for continuous writes, and deduplication runs later as its own analytical step.

## Why deduplication happens before classification

Classification reads the deduplicated table, not the raw one:

```text
Raw transaction records
          │
          ▼
      Deduplication
          │
          ▼
Deduplicated transactions
          │
          ▼
       Classification
```

Classification assigns ranks and moves cumulative user state. If one payment still had two physical rows, both could earn classification positions and trigger two rule evaluations, two state transitions, two lifetime count increments, two amount increments, and two vendor events. The deduplication boundary guards downstream state, not just data quality.

## Inside-window deduplication

Within the current window, the three tiers apply in order. Rows inside each tier rank by ingestion time:

```sql
ROW_NUMBER() OVER (
    ...
    ORDER BY ingested_at ASC
)
```

The first arrival survives:

```text
Same identity
    │
    ├── Record A → ingested first → KEEP
    ├── Record B → later          → DROP
    └── Record C → later          → DROP
```

Window bounds use `ingested_at`, the same clock the orchestrator classifies on.

## Cross-window deduplication

The harder case is a duplicate that lands after its original window already closed. Here incoming records get compared against records that already survived in the deduplicated table, through three anti-joins in identity order:

```text
Incoming record
      │
      ├── Match payment-group identity? ──► duplicate
      │
      ├── Match payment reference? ───────► duplicate
      │
      └── Match transaction identity? ─────► duplicate
```

A record gets inserted only when no historical identity matches. Every match stays scoped to the same customer, and a tier joins the matching only when its identity field is actually present. Missing identifiers can never match unrelated history.

## Why the historical lookback is limited

The cross-window anti-join scans a bounded date range around the incoming transaction's own transaction date:

```sql
historical_txn_date
    BETWEEN incoming_txn_date - N days
        AND incoming_txn_date + N days
```

The implementation uses three days on either side. Anchoring to the incoming record's transaction date instead of today's date means a delayed representation still meets its earlier counterpart days later.

Records whose transaction date falls outside the band can slip through. That is an explicit exchange of coverage for scan cost.

## Late-arriving records

Take one payment with two upstream records:

```text
Window A
   └── Record A arrives
          ↓
      survives dedup
          ↓
       classified

Window B
   └── Record B arrives later
          ↓
      historical anti-join
          ↓
      matches Record A
          ↓
        dropped
```

Deduplication reaches past the current window. The historical anti-join lets a new arrival meet records accepted earlier.

## Why ingestion does not solve this problem

Pushing more filtering into the Kafka ingestion layer looks tempting until an identity field turns out to be optional. Legitimate payments can arrive without the higher identifiers, and filtering at ingestion would delete valid transactions before their fallback identity ever got evaluated.

So each layer answers one question. Ingestion asks whether a record is valid enough to enter the analytical system. Deduplication asks whether it represents a payment already accepted:

> Is this record valid enough to enter the analytical system?

> Does this record represent a payment we have already accepted?

## Why the work stays inside StarRocks

Raw and deduplicated data already sit in the same cluster. The historical step anti-joins the incoming window against accepted records, and moving millions of rows out to the Go orchestrator for that would be wasteful:

```text
Kafka
  │
  ▼
StarRocks
  │
  ├── Raw transaction table
  │
  └── Deduplicated transaction table
          │
          ▼
      Classification
```

The orchestrator triggers the SQL and tracks its status. It does not process data.

At documented scale that means about 90 to 100 million raw successful and debit records a day, roughly 60 to 70 million after dedup, with the dedup step itself taking seconds under the tested workload.

## Data layout and locality

Both tables share the same physical strategy:

- daily partitioning on ingestion time;
- hash distribution by customer;
- columnar compression;
- a bloom filter on customer identity.

Customer distribution pays off because dedup joins are customer scoped. Most comparison work stays local to one backend distribution instead of reshuffling across the cluster, and the historical side reads only the needed ingestion partitions while the bounded transaction-date condition narrows the anti-join.

## What the classification stage assumes

After dedup, classification treats the window as unique accepted payments. Records order by transaction time with a deterministic transaction identifier breaking ties, so a removed physical duplicate never earns its own rank. Cumulative state updates cannot safely run twice, which is why this guarantee matters:

```text
Deduped payment
      ↓
one classification row
      ↓
one state increment
      ↓
one downstream event
```

instead of:

```text
Duplicate payment
      ↓
two classification rows
      ↓
two state increments
      ↓
two downstream events
```

## Design trade-offs

Four calls shaped this design.

### 1. Append-oriented ingestion

The raw table never enforces global uniqueness while ingesting. Writes stay simple and fast.

### 2. Multi-tier identity

Several identity levels describe messy transaction representations better than one key. The price is more complex SQL and extra anti-join work.

### 3. Bounded historical lookback

Cross-window search covers limited history. Cost stays controlled while the expected late arrivals still match.

### 4. First arrival wins

Inside one window, the earliest ingested representation survives. Behavior stays deterministic with no second ranking rule for ingestion order.

## A useful mental model

Think of the dedup layer as a payment identity resolver sitting between ingestion and classification:

```text
              Continuous Kafka Ingestion
                         │
                         ▼
                Raw Transaction Store
                         │
                         ▼
              ┌─────────────────────┐
              │     Deduplication   │
              │                     │
              │  1. Group identity  │
              │  2. Payment ref     │
              │  3. Transaction ID  │
              │                     │
              │  + historical check │
              └──────────┬──────────┘
                         │
                         ▼
                Accepted Payments
                         │
                         ▼
                   Classification
```

Any event stream with redeliveries, multiple representations of one business event, incomplete identifiers, delayed arrivals, or stateful downstream work benefits from an identity layer with a clear correctness boundary instead of pushing every case into ingestion.

## Closing thought

High-volume deduplication is not:

```sql
SELECT DISTINCT ...
```

Once an event stream carries several representations of one business event, dedup turns into an identity problem:

```text
What identifies the business event?
        ↓
Which identity is authoritative?
        ↓
What happens when an identifier is missing?
        ↓
Can the same event arrive again later?
        ↓
How far back should historical matching go?
        ↓
What downstream state becomes incorrect if deduplication fails?
```

Settle those first. The SQL gets easier to reason about, and the pipeline holds up.

*Next: state management, snapshots, and recovery. Till then enjoy your life and happy engineering!*
