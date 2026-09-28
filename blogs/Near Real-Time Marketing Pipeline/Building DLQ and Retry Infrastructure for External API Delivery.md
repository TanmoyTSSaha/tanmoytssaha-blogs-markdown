---
title: Building DLQ and Retry Infrastructure for External API Delivery
slug: dlq-retry-external-delivery
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - reliability
  - dlq
  - real-time
description: How failed vendor deliveries split into immediate retries, durable DLQ records, and scheduled recovery without stalling the pipeline.
reading_time: 18
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 10
---
HTTP integrations fail, and a failed event should not freeze the whole pipeline. This design splits delivery problems into three tracks that run apart:
```text
Normal delivery
     +
Immediate retries
     +
Durable failure handling
```
Once a worker's local retry budget runs out, the event becomes a dead letter. A separate retry process later decides whether it qualifies for automatic recovery, so the original consumer never sits blocked.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=2VRsxfthdjakJvsMIdMq">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=2VRsxfthdjakJvsMIdMq&type=embed" /></a>

## The path

```text
Vendor Delivery Worker
          │
          ▼
      External API
          │
   ┌──────┼─────────────┐
   │      │             │
success  retryable    permanent
   │      │             │
   ▼      ▼             ▼
offset   retry       DLQ record
commit   same batch      │
                         ▼
                  Retry Scheduler
                         │
                         ▼
                 Canonical Kafka Topic
                         │
                         ▼
                  Enrichment / Routing
                         │
                         ▼
                  Vendor-specific topic
                         │
                         ▼
                  Vendor Worker
```

Retries re-enter the normal event flow instead of skipping routing and enrichment.

## Why failures are split

Every HTTP failure does not deserve the same response:

```text
Transient failure
      ↓
Retry

Recoverable later
      ↓
Persist in DLQ
      ↓
Automatic retry

Permanent failure
      ↓
Persist for investigation
      ↓
Do not automatically retry
```

That split steers between the two bad defaults:

```text
Retry everything forever
```

```text
Drop everything that fails once
```

The implementation keeps retryable, permanent, and legacy retryable failure reasons apart.

## Immediate retries inside the vendor worker

Before anything becomes a DLQ record, the vendor worker retries transient failures in place:

```text
Kafka batch
    │
    ▼
HTTP call
    │
    ├── success ──► commit offset
    │
    └── retryable ──► wait ──► same batch
```

Short outages clear in seconds, and handling them inline avoids writing a durable retry record for trouble that disappears on its own. The wait can come from the error itself:

```text
Rate limiting
      ↓
Server-provided / configured delay

Temporary server error
      ↓
Short retry delay

Circuit breaker open
      ↓
Wait for dependency recovery
```

Exact delays and attempt counts stay deployment-specific and out of this article.

## Why not retry indefinitely in the consumer

One stuck batch poisons everything behind it:

```text
One bad batch
    ↓
Consumer stuck
    ↓
Partition stops progressing
    ↓
Later events wait behind it
```

So local retries stop once their budget runs out. A still retryable failure converts into a durable DLQ record, the Kafka offset moves on, and the pipeline's blocking period stays bounded.

## Permanent versus retryable failures

### Retryable

Temporary server failures and similar conditions where waiting may fix things:

```text
HTTP 5xx
temporary dependency failure
temporary infrastructure problem
```

### Permanent

Malformed payloads, bad configuration, or requests the destination explicitly refuses:

```text
invalid request
invalid payload construction
invalid URL/configuration
non-retryable authentication/configuration failure
```

Permanent records stay available for investigation and never enter automatic retry. Exact status lists and internal reason codes stay out of this write-up.

## Why a DLQ needs two representations

Dead-letter information lives in two places:

```text
                Failed Event
                     │
             ┌───────┴────────┐
             ▼                ▼
        Kafka DLQ         Durable DLQ Store
       forensics copy      retry work queue
```

### Kafka dead-letter stream

An immutable long-lived event record with what debugging and forensics need.

### Durable database queue

What the retry process queries efficiently:

- which records are active;
- how many automatic retries ran;
- when each record becomes retryable again;
- whether it resolved; and
- whether it turned terminal.

Kafka keeps the event history. The database runs the retry queue.

## DLQ lifecycle

A DLQ record typically moves:

```text
active
  │
  ▼
retrying
  │
  ├── success ──► resolved
  │
  └── retryable failure ──► active
                              │
                              ▼
                           retry again

active
  │
  ▼
retry cap reached
  │
  ▼
exhausted
```

Permanent failures go terminal immediately:

```text
permanent
```

Automatic retry never selects terminal states.

## Making the DLQ record unique

One logical event can fail per destination during recovery, so the store draws its uniqueness line at:

```text
Event identity + target destination
```

One canonical event fanning out looks like:

```text
Event X
 ├── Destination A → DLQ row A
 └── Destination B → DLQ row B
```

Destination A's failure state never overwrites Destination B's.

## The retry scheduler

A separate scheduled process scans the durable queue:

```text
Find active records
       │
       ▼
Check retry count
       │
       ▼
Check waiting period
       │
       ▼
Claim eligible record
       │
       ▼
Publish back to canonical Kafka flow
```

Scheduling lives apart from delivery, so vendor workers carry no long-term retry timers.

## Selecting eligible retries

A retry runs only when everything holds:

```text
status = active
        AND
retry count < maximum
        AND
waiting period elapsed
        AND
failure reason is retryable
```

Older records go first. Deterministic selection, and permanent failures never enter the loop.

## Claiming a retry safely

Before publishing, the scheduler flips the record from active to retrying and bumps the automatic retry count. The update only applies while the record is still active:

```text
active
  │
  │ atomic claim
  ▼
retrying
  │
  ▼
publish
```

Two retry workers cannot claim the same row. When the Kafka publish fails, the claim rolls back:

```text
retrying
    │
    X publish failure
    │
    ▼
active
```

The budget restores because the event never re-entered normal flow.

## Why retries re-enter the normal pipeline

The retry path never calls the external API directly:

```text
DLQ
 ↓
Canonical Kafka Topic
 ↓
Enrichment
 ↓
Routing
 ↓
Vendor Topic
 ↓
Vendor Worker
 ↓
External API
```

No second vendor-delivery implementation exists. Normal and recovery traffic share enrichment, routing, payload generation, and delivery logic, so the two behaviors cannot drift apart.

## Re-enrichment on retry

Retries do not freeze the identifiers from the original attempt. Each retry rebuilds a canonical event and walks the normal enrichment path:

```text
DLQ record
   ↓
Canonical event
   ↓
Lookup latest identifiers
   ↓
Routing
   ↓
Vendor-specific event
```

Identifiers and routing configuration may have changed since the first attempt, and the retry picks up the current values. Internal metadata marking auto-retried events stays out of this write-up.

## Why the retry path can fan out again

Retries return to canonical routing instead of aiming at the original vendor, so current routing rules evaluate fresh:

```text
Retry
  ↓
Canonical event
  ↓
Current routing rules
  ├──► Destination A
  ├──► Destination B
  └──► Destination C
```

The owning worker settles its destination's DLQ state once the external API succeeds. The retry mechanism stays generic by design.

## Exponential backoff versus scheduled retries

Two retry mechanisms cover different ground.

### Immediate worker retry

Short failures during the original attempt:

```text
Failure
  ↓
wait
  ↓
retry same batch
```

Fixed delays or error-derived waits.

### Durable DLQ retry

After local attempts stop:

```text
DLQ
  ↓
wait
  ↓
retry scheduler
  ↓
canonical Kafka flow
```

Its own cap and waiting interval. The two counters must never merge into one global retry count.

## Avoiding infinite retry loops

Four escape points keep retries finite.

### 1. Permanent failures

Automatic retry queries never select permanent records.

### 2. Retry cap

Retryable records eventually exhaust their automatic count:

```text
active
  ↓
retry
  ↓
retry
  ↓
retry
  ↓
exhausted
```

### 3. Broken Kafka publish

A retry that cannot even reach the canonical flow returns to active for later instead of burning budget permanently.

### 4. Vendor-side worker limits

Vendor workers run their own short-term retry policy, kept deliberately apart from the DLQ budget:

```text
Worker retry budget
        +
DLQ retry budget
```

## Immediate retry versus durable retry

```text
          Vendor Failure
               │
       ┌───────┴────────┐
       │                │
   Short-lived       Needs later
    problem           recovery
       │                │
       ▼                ▼
In-process retry       DLQ
       │                │
       └───────┬────────┘
               ▼
           Final outcome
```

Short retries keep transient noise out of the DLQ. Durable retries recover failures that outlive the original consumer attempt.

## Isolating destinations

Every destination owns a worker and a topic:

```text
                    Canonical Events
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        Destination A  Destination B  Destination C
             │            │            │
          Worker A     Worker B     Worker C
             │            │            │
             ▼            ▼            ▼
          External      External      External
             APIs          APIs          APIs
```

Destination A failing never blocks B or C, and the shared retry system still tracks each DLQ row independently through the destination in its identity.

## Observability

DLQ state has to answer:

```text
What failed?
Where did it fail?
When did it fail?
How many automatic retries happened?
When will it be eligible again?
Was it eventually resolved?
Was it permanently rejected?
```

Working states look like:

```text
active
retrying
resolved
exhausted
permanent
```

Retry counts and timestamps separate fresh failures from old events on their tenth attempt. Metric names, dashboards, and alert ids stay out of this write-up.

## Forensics versus work queue

The Kafka DLQ copy and the database queue answer different questions:

```text
Kafka DLQ
   ↓
Historical investigation
   ↓
What actually failed?

Database DLQ
   ↓
Operational scheduling
   ↓
What should be retried now?
```

One storage technology never serves both access patterns well.

## Replaying a resolved event

A resolved record succeeded eventually, but operators sometimes still need a replay for exceptional cases. Replay stays a delivery operation, never a new classification:

```text
Do not re-run stateful classification
just to resend an already-classified event.
```

Delivery replays against the event layer while committed user state stands untouched.

## Design trade-offs

### 1. Immediate retries versus DLQ

Immediate retries answer transient failures fast but stall a consumer when overused. The DLQ adds durability with delay.

### 2. Database queue versus Kafka-only retry

Kafka moves events well. A database filters and schedules over retry metadata well. Each serves what it suits.

### 3. Retry budget versus recovery probability

Bigger budgets recover more transient failures while burning resources and postponing terminal calls. The cap is operations policy, not a universal number.

### 4. Re-entering the canonical pipeline versus direct vendor retry

Normal-flow re-entry avoids duplicate implementations, at the cost of re-evaluating routing and enrichment. Consistency wins deliberately.

### 5. Per-destination isolation versus service count

Independent vendor workers isolate failures and multiply running services. The payoff grows with every integration added.

## A useful mental model

Delivery and recovery form two layers:

```text
                 DELIVERY LAYER
                       │
                       ▼
                 External API
                       │
                failure / retry
                       │
                       ▼

                 RECOVERY LAYER
                       │
                       ▼
                      DLQ
                       │
                 retry scheduler
                       │
                       ▼
              Canonical Kafka Flow
                       │
                       ▼
                 Delivery Layer
```

Recovery never becomes a second delivery implementation. Failed events feed back into the path normal events take.

## Closing thought

External integrations fail sometimes. The engineering target reads less like preventing failure and more like handling it in order:

```text
Detect
  ↓
Classify
  ↓
Retry when appropriate
  ↓
Persist when recovery is delayed
  ↓
Stop retrying when the event is terminal
  ↓
Preserve enough information for investigation
```

Short-lived retries answer transient trouble fast. A durable DLQ holds failures that need later attention. A scheduler returns recoverable events to normal flow. Together they recover in a controlled way while one failing dependency never stalls the pipeline.

*That closes the delivery arc. Till then enjoy your life and happy engineering!*
