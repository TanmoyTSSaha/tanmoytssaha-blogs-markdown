---
title: Processing a Classification Window at Scale
slug: processing-classification-window
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - starrocks
  - classification
  - real-time
description: "Inside one classification window - the six stages, the execution ledger, and how the pipeline recovers without corrupting user state."
reading_time: 14
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 3
---
The first post sketched the pipeline and the second covered ingestion. This one goes inside a classification window: the six stages it passes through, and how each stage is built to fail safely.
> **Important:** This article uses a 30-minute window as the running example. Production currently classifies hourly. The stages are identical either way, and the pipeline supports sub-hourly intervals without changing them.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=hGqAbBBwlesmAPmZFwQm">View on Eraser<br /><img src="https://raw.githubusercontent.com/TanmoyTSSaha/tanmoytssaha-blogs-markdown/main/blogs/Near%20Real-Time%20Marketing%20Pipeline/assets/hld-03-window.png" /></a>

## The path

The order is fixed. Publishing comes last, and a newer window does not start until the current one has finished or failed.

## Why a CronJob starts the pipeline

The orchestrator is a short lived process, not a long running Kafka consumer. A Kubernetes CronJob wakes it up on a period and asks whether any new classification window is eligible:

```text
Scheduler wake       → asks "Is there work?"
Classification window → defines "What data should be processed?"
```

A wake can finish with nothing done. There may be no closed window yet, another run may hold the pipeline lock, or a previous run may still be going. When a run genuinely fails, it gets marked failed so the same window can be retried later.

## Eligibility of a classification window

A window runs only after it has closed and a short settlement period has passed. For a 30-minute interval:

```text
10:00 ─────────────── 10:30 ───── 10:35
        window              settle
        closes              period
```

At 10:35, the `10:00–10:30` slice is eligible. Ingestion never pauses, so data keeps arriving near the boundary. The settlement gap gives the ingestion layer time to commit those rows before the closed slice gets classified.

## One window is an ingestion time slice

Window boundaries use the timestamp assigned when the record landed in the analytical layer, not the original transaction timestamp:

```sql
WHERE ingested_at >= window_start
  AND ingested_at <  window_end
```

A transaction that happened earlier but arrived late belongs to the ingestion window where it first became visible. A record that lands after its window closed belongs to a later one. So a window is a closed block of arrivals, never a rolling `now - N minutes` query.

## Processing backlog safely

The orchestrator remembers the latest window that reached publish. When it falls behind, it does not jump to the newest window:

```text
Oldest incomplete window
          ↓
Process
          ↓
Next window
          ↓
Process
          ↓
Next window
```

Oldest first is a correctness call. Classification is stateful, and skipping ahead would apply state transitions out of order. Catch-up work per wake is also bounded, so a deep backlog cannot turn one run into an unbounded execution.

## Why two windows cannot run at once

Four layers keep execution to one window at a time.

### 1. Kubernetes scheduling

The CronJob does not start a new pod while the previous scheduled run is still active.

### 2. Database level lock

The orchestrator takes a short lived database lock before reading or changing pipeline state. A second process that misses the lock exits instead of queueing behind the active run.

### 3. Running state check

The pipeline also looks in its execution ledger for an already running window. That covers executions that began before the lock existed.

### 4. Sequential processing inside one run

Eligible windows run oldest first in a loop:

```text
Window A
   ↓
finish / fail
   ↓
Window B
   ↓
finish / fail
   ↓
Window C
```

A later window waits while the earlier one executes, and a failed window blocks everything behind it until it recovers.

## What happens inside one window

Six stages, each with its own success condition:

| Stage | Purpose | Success condition |
|---|---|---|
| Configuration Sync | Copies the current classification configuration from the relational configuration store into the analytical processing layer. | A complete rule set is available for classification. |
| Deduplication | Reduces duplicate records from the closed ingestion slice. | The deduplicated records for the window are available. |
| Classification | Applies mutually exclusive classification rules to the users in the window. | Classified users and the event delta are produced. |
| State Snapshot | Saves the previous user state before modification. | A recovery image exists for affected users. |
| State Update | Applies the new classification state and lifetime counters. | Current state reflects the completed classification window. |
| Publish | Sends the resulting event delta to Kafka and records publish progress. | The window has been successfully published. |

The execution ledger records every stage, so a wake skips stages that already succeeded and a failed window resumes from its last good boundary.

## Stage 1: configuration sync

Classification rules live outside the application code. Each window starts with a full copy of the active rule set synced from the relational store into the analytical layer.

The pipeline copies the full active rule set before classifying. Rules can change through the configuration interface without a redeploy, and each window sees one consistent snapshot instead of catching a rule edit mid flight. When the sync fails, classification does not start.

## Stage 2: deduplication

Classification should read a stable view of the window's transactions. Deduplication collapses repeat representations of a payment through identity levels, falling back to progressively broader identifiers:

```text
Primary identity
      ↓
Fallback identity
      ↓
Transaction identity
```

Duplicates are deliberately not stopped at Kafka ingestion. The raw table stays tuned for continuous writes, and correctness gets handled here as its own analytical step:

```text
Write-optimized ingestion
```

```text
Correctness-oriented analytical processing
```

## Stage 3: classification

This is the core of the pipeline. The orchestrator builds classification queries from the current rule configuration and runs them over the window's users. Classification is mutually exclusive: each user ends the window in one appropriate state, not scattered across competing ones. Existing user state and the transaction history available to the analytical layer feed into the decision.

The latest record per user inside the window drives the outcome, since that record decides the state applied for this window.

## Stage 4: state snapshot

Classification rewrites user state, so the old state gets protected first in a snapshot that acts as a recovery point.

The snapshot is a recovery point. When the update that follows fails halfway, the pipeline restores the previous state instead of layering another update over possibly corrupt data. That matters because the state table holds cumulative information, not disposable output.

## Stage 5: state update

With the snapshot stored, the newly classified users go into current state.

Updates carry the current classification plus lifetime counters and other state built up by earlier windows. Since the updates accumulate, running the same one twice is unsafe, which is why the pipeline tracks explicit stage state alongside the pre-update snapshot.

## Stage 6: publish

After state updates land, the event delta goes to Kafka in chunks with progress persisted.

Delivery here is at least once. A crash after Kafka acknowledges a chunk but before the checkpoint writes means that chunk goes out again on recovery, so downstream has to tolerate replays.

## Progress tracking

The orchestrator keeps an execution ledger with every stage of every window:

Each entry carries current status, attempt info, start and completion timestamps, failure details, heartbeat data, and the latest published window. The ledger is what turns a string of SQL steps into a workflow that can resume.

## Heartbeats and stale run recovery

A long stage keeps updating its execution state, so the next run can tell an active process from a dead one. When a process dies mid query, the following run spots the stale execution and cleans up before resuming. Exact thresholds stay in deployment configuration and out of this write-up.

## Failure boundaries

Retries happen per stage:

The next wake resumes at the failed stage instead of replaying the ones that already passed.

### Failure between snapshot and state update

This is the recovery case the whole snapshot stage exists for:

```text
State Snapshot
      │
      ▼
Backup exists
      │
      ▼
State Update fails
      │
      ▼
Restore previous state
      │
      ▼
Retry classification / state update
```

The pipeline restores the previous state and rebuilds the window from known ground. It never applies the update again over partially written state.

### Failure during publishing

Publishing fails differently:

```text
Kafka write succeeds
        ↓
Process crashes
        ↓
Publish checkpoint not written
```

On restart the chunk goes out again. The system prefers at least once delivery over quietly dropping an event, and the coverage watermark stays put until the final publish stage actually succeeds.

## Why a failed window blocks newer windows

A failed window never gets skipped:

```text
Window A → FAILED
Window B → available
Window C → available
```

Recovery runs A, then B, then C. Letting B and C through first would let later windows rewrite user state before the earlier one completed, scrambling the history. Sequential recovery keeps the user state machine in order.

## A useful mental model

Three clocks run the design:

```text
1. Ingestion clock
   Kafka continuously delivers data.

2. Window clock
   The orchestrator closes and processes discrete ranges of arrivals.

3. State clock
   User state advances only after the previous window completes successfully.
```

Ingestion stays always on while classification works through deterministic, recoverable windows.

## Design lessons worth reusing

These travel well beyond marketing systems.

### Separate ingestion from processing

Continuous ingestion and windowed processing answer different questions. Forcing them onto one clock makes boundaries and recovery harder than they need to be.

### Keep configuration outside the processing code

A relational configuration store lets business rules change without rebuilding and redeploying the service.

### Treat state updates as a protected operation

When classification rewrites durable user state, a pre-update recovery point stops partial failures from corrupting it permanently.

### Make progress explicit

An execution ledger turns processing steps into a workflow that can resume.

### Assume replay can happen

At least once publishing means downstream systems handle duplicate delivery instead of depending on exactly once.

### Preserve ordering for stateful workflows

When one window's output seeds the next window's starting state, sequential processing is a correctness requirement.

## Closing thought

A near real time pipeline is more than a batch job run often. The actual work sits in coordinating continuous ingestion, closed windows, stateful classification, recovery, and replay safe publishing:

```text
continuous ingestion
       +
closed processing windows
       +
stateful classification
       +
failure recovery
       +
replay-safe publishing
```

Designed apart, each concern stays easy to reason about and to bring back up.

*Next: enrichment, vendor routing, delivery workers, and the DLQ. Till then enjoy your life and happy engineering!*
