---
title: At-Least-Once Delivery
slug: at-least-once-delivery-publish-progress
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - delivery
  - reliability
  - real-time
description: "How classified events reach Kafka exactly as reliably as needed - checkpoints, keyset pagination, deterministic event ids, and replay-safe consumers."
reading_time: 16
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 8
---
Moving classified events from the analytical platform into Kafka raises a different correctness question than classification. State transitions apply once. Delivery tolerates a message Kafka accepted while the publisher died before recording it:
```text
Processing correctness
        +
Delivery reliability
```
The publish stage carries an explicit progress checkpoint. It resumes from the last recorded position and accepts a small replay window.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=1LLwBE-ClUoc1azjTujh">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=1LLwBE-ClUoc1azjTujh&type=embed" /></a>

## Two different guarantees

### Processing correctness

User state changes during state update. Once that succeeds, replaying a Kafka message never runs the state update again:

```text
Classification
      ↓
State Update
      ↓
Successful state transition
      ↓
Publish
```

A publish replay creates no new classification and no new cumulative update.

### Delivery guarantee

The Kafka hop runs at least once delivery. The system would rather duplicate a message than silently lose one. The failure case that shapes everything:

```text
Kafka accepts message
        ↓
Checkpoint fails / process crashes
        ↓
Publisher resumes from previous checkpoint
        ↓
Same message is written again
```

Duplicates are expected. Downstream handles them.

## Why exactly once delivery is hard at this boundary

The publishing service eventually feeds downstream HTTP integrations:

```text
Publish to Kafka
      ↓
Consumer receives message
      ↓
HTTP request accepted
      ↓
Consumer crashes
      ↓
Kafka offset was not committed
```

On restart the same Kafka message gets consumed again, and the external platform may see the same logical event twice. So the architecture puts idempotency on the consumer side instead of chasing exactly once across the whole Kafka to HTTP chain. Event identity stays deterministic so replays stay recognizable.

## What sits between the analytical layer and Kafka

The publisher reads one window's event delta and sends it to Kafka:

```text
Enriched Event Delta
        │
        ▼
    Read in Pages
        │
        ▼
   Split into Chunks
        │
        ▼
      Kafka
        │
        ▼
Persist Publish Progress
```

Big windows never load fully into application memory. The reader pages through data while Kafka messages go out in smaller chunks, which bounds the working set and draws natural checkpoint lines.

## Why pagination matters

One window can hold millions of output events. Loading everything at once burns memory and makes restarts harder to think through. The publisher pages, chunks, sends, and checkpoints:

```text
Page
  ↓
Chunk
  ↓
Kafka
  ↓
Checkpoint
```

Results come back in deterministic order. Resume starts from the last checkpointed record, not from a count of what already went out.

## Why a count is not enough

```text
events published = 20,000
```

That number cannot say where to resume. Physical layout shifts, and several rows can share individual sort fields. The checkpoint stores the logical position of the last accepted row as a multi-column cursor:

```text
(last customer,
 last event type,
 last event timestamp,
 last transaction id)
```

The next query asks for rows strictly past that position. That is keyset pagination.

## Keyset pagination versus OFFSET

Offsets look easy at first:

```sql
LIMIT 2000 OFFSET 1000000
```

But the database still walks the preceding million rows to reach the page. On huge result sets that walk gets slower the further publishing gets, and restarts pay the same cost again. Keyset pagination asks a different question:

> "Skip the first N rows."

becomes:

> "Give me rows after this exact last-known row."

```text
Last checkpoint
      │
      ▼
Rows after checkpoint
      │
      ▼
Next chunk
```

Restarts stay predictable no matter how large the window grows.

## The publish progress record

Each classification window owns one logical progress record:

```text
window identifier
events published
last customer
last event type
last event timestamp
last transaction id
publish status
```

It answers one question:

> How far did this window get through the Kafka publishing stage?

The pipeline execution ledger answers a different one:

```text
Pipeline execution ledger
        ↓
Did the PUBLISH stage succeed?

Publish progress
        ↓
Which row should PUBLISH resume from?
```

Processing state and delivery progress stay in separate books.

## How the checkpoint moves

One publish cycle:

```text
1. Read next page
2. Build Kafka messages
3. Send chunk
4. Kafka acknowledges
5. Update checkpoint
6. Continue
```

The ordering that matters:

```text
Kafka acknowledgement
        ↓
Checkpoint
```

never the reverse. Writing the checkpoint before Kafka acknowledges could move the cursor past messages nobody delivered. That is data loss.

## The replay gap

Ack-first ordering leaves a small deliberate replay gap:

```text
Message chunk
     │
     ▼
Kafka accepts
     │
     ▼
Checkpoint update
     X
 process crashes
```

The messages sit in Kafka while the checkpoint still points at the previous chunk. On resume the same chunk gets selected again with the same event ids. The system replays instead of losing data.

## Failed Kafka publish

The mirror case matters too. When Kafka never acknowledges:

```text
Build chunk
    ↓
Kafka publish fails
    ↓
No checkpoint
    ↓
Retry from last valid checkpoint
```

Unacknowledged chunks never count as delivered, so the checkpoint keeps tracking the confirmed Kafka boundary.

## What happens when a checkpoint update fails

A checkpoint write can fail on its own after Kafka accepted the chunk:

```text
Kafka = accepted
Checkpoint = old
```

The next attempt resends the chunk. Resending beats the alternative: a checkpoint advanced without confirmation skips records permanently.

## Why a replay does not change user state

Publishing runs after state update:

```text
Classification
      ↓
State Update
      ↓
Publish
```

Retrying publish never re-runs the state update. State machine and delivery pipeline stay logically apart.

## Deterministic event identity

Every published event carries an identifier derived from the business identity behind it:

```text
user identity
    +
event type
    +
classification window
    +
transaction identity
    ↓
Deterministic Event ID
```

The same source row produces the same identifier on every replay. Random ids per retry would be far less useful:

```text
Original delivery → random ID A
Replay             → random ID B
```

Downstream could never tell those apart. Deterministic ids fix that:

```text
Original delivery → Event ID X
Replay             → Event ID X
```

Duplicates become recognizable.

## Why transaction identity is part of the event identity

One user can legitimately produce several transactions of the same event type inside one window, so user plus event type plus window is not unique:

```text
user + event type + window
```

Transaction identity separates those independent events while keeping replays distinct from genuine duplicates:

```text
Same event replayed
```

```text
Two legitimate events for the same user
```

## Downstream idempotency

Past the consumers, the deterministic identifier carries duplicate handling. A consumer can keep a short lived in-process cache of recent event ids to skip repeating an external HTTP call during an immediate replay. That cache dies with the process:

```text
Process restart
      ↓
Cache lost
      ↓
Duplicate event can reach HTTP again
```

Durable idempotency ultimately rests on the external integration, or another persistent mechanism, recognizing the deterministic identity. The standing rule:

> At-least-once delivery requires replay-safe consumers.

## Retry, resume, and replay are different operations

Three words that should never blur together.

### Resume

Keep publishing from the existing checkpoint:

```text
Existing progress
      ↓
Continue after cursor
```

### Retry

Re-run an incomplete publish stage:

```text
PUBLISH failed
      ↓
Read checkpoint
      ↓
Continue
```

### Replay

Deliberately resend a finished window:

```text
Operator requests replay
      ↓
Clear publish checkpoint
      ↓
Start from beginning of window
```

Replay touches delivery only. Committed user classification state stays as it is.

## Why this separation matters

A finished window looks like:

```text
Classification      ✓
State Update        ✓
Publish             ✓
```

When operators later decide delivery needs regenerating, the safe move is replaying publish, not rerunning classification and state update on top of it. Reapplying cumulative changes would double count the window. Separate ledgers for processing state and delivery progress make that distinction enforceable.

## What happens to skipped records

The cursor advances only to the last record inside an accepted Kafka chunk. A record that breaks while being read or converted never joins the chunk. Malformed or unprocessable records need their own handling and monitoring instead of passing silently as delivered. Internal logging fields and error-handling identifiers stay out of this write-up.

## End to end view

```text
                 Classification
                       │
                       ▼
              Enriched Event Delta
                       │
                       ▼
                  Pagination
                       │
                       ▼
                    Chunk
                       │
                       ▼
                    Kafka
                       │
             ┌─────────┴─────────┐
             │                   │
          Acked              Not Acked
             │                   │
             ▼                   ▼
        Checkpoint            Retry
             │
             ▼
       Next Chunk
```

One boundary governs failure:

```text
Kafka acknowledgement
        ↓
Checkpoint
```

Anything failing before the checkpoint stays retryable. Anything failing after Kafka's acknowledgement but before the checkpoint may arrive twice.

## Design trade-offs

### 1. At-least-once versus exactly-once

At-least-once stays simpler across mixed systems, especially with an external HTTP API at the end. Consumers accept duplicates in return.

### 2. Checkpoint after acknowledgement

Ack-first ordering removes silent loss and opens a replay window. Deliberate.

### 3. Keyset pagination versus OFFSET

Keyset restarts stay fast and deterministic on huge result sets, at the cost of needing a suitable ordered cursor.

### 4. In-memory duplicate cache versus durable idempotency

An in-process cache suppresses immediate duplicate HTTP calls cheaply and vanishes on restart. Durable correctness needs the event identity recognized downstream.

### 5. Separate processing and delivery ledgers

Extra bookkeeping, and delivery retries can never reapply stateful classification by accident.

## A useful mental model

The publishing layer works as a durable handoff cursor:

```text
                 Event Stream
                     │
                     ▼
             ┌───────────────┐
             │   Publisher   │
             └───────┬───────┘
                     │
                 Chunk
                     │
                     ▼
                   Kafka
                     │
                ACK received
                     │
                     ▼
              Save cursor
                     │
                     ▼
             Read next chunk
```

The cursor never promises everything behind it exists exactly once. It promises everything behind it reached Kafka with progress recorded. That distinction carries the whole at-least-once design.

## Closing thought

Reliable delivery rarely means making every component exactly once. The practical version draws explicit lines:

```text
State transition
      ↓
Apply once

Message delivery
      ↓
Allow replay

Event identity
      ↓
Make replay recognizable

Progress checkpoint
      ↓
Resume safely
```

At-least-once delivery is a system design across checkpoint ordering, deterministic identity, replay-safe consumers, separate processing and delivery state, and explicit replay semantics. Once those lines hold, a crash between Kafka acceptance and progress recording turns into a controlled duplicate instead of uncontrolled loss.

*A crash between accept and checkpoint is where most delivery bugs hide. Worth a post of its own sometime. Till then enjoy your life and happy engineering!*
