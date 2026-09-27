---
title: Stateful User Classification
slug: stateful-user-classification
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - starrocks
  - state-machine
  - recovery
  - real-time
description: How the pipeline snapshots user state before updating it, and why cumulative counters make blind retries unsafe.
reading_time: 16
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 6
---

One window's transactions alone cannot decide a user's next state. The answer depends on what the previous window left behind, which makes this pipeline stateful:

```text
Previous User State
        +
Current Window Transactions
        ↓
New User State
```

Run a state update twice, or let two windows touch state together, and cumulative counters go wrong. The pipeline guards against both by protecting current state before changing it and keeping enough around to restore it on failure.

> **Important:** Production classifies hourly. The stages below work the same at shorter intervals such as 30 minutes.

## The tables

State handling boils down to this flow:

```text
User State
   │
   │ read during classification
   ▼
Temporary Classification Results
   │
   ├───────────────┐
   │               │
   ▼               ▼
State Snapshot   State Update
   │               │
   ▼               ▼
State Backup    Current User State
```

Three datasets take part.

### 1. Current user state

The live state per `(user, journey)` pair. It holds whatever the next window needs:

- current classification state;
- when the user entered that state;
- transaction count in the current state;
- latest transaction date;
- first transaction date;
- lifetime transaction count;
- lifetime transaction amount.

### 2. Temporary classification results

The winning decisions for the current window. A staging area, not a backup of anything.

### 3. State preimage / backup

The previous state of exactly the user-journey pairs this window is about to change. An undo log.

## Why the classifier has to remember

Say a user enters a state in Window A. In Window B the next transaction forces a choice:

```text
stays in the same state
          OR
moves to another state
```

That choice can rest on history:

```text
Current state
Transaction count in state
Days spent in state
Days since last transaction
Lifetime transaction count
Lifetime amount
```

None of those belong to the current payment. Earlier windows produced them, so classification reads as:

```text
Classification(Window N)
        =
Transactions(Window N)
+
State(Window N-1)
```

## Missing state means a new user

A `(user, journey)` pair with no state row starts from initial state with zeroed counters:

```text
No previous state
      ↓
Initial state
      ↓
Evaluate current transaction
      ↓
Determine new state
```

One engine handles first sightings and long accumulated histories alike:

- users being seen for the first time; and
- users whose state has already been accumulated across many windows.

## Multiple journeys have independent state

One user can walk more than one journey:

```text
                    User
                     │
            ┌────────┴────────┐
            ▼                 ▼
      Overall Journey    Vertical Journey
            │                 │
            ▼                 ▼
       Current State      Current State
       Counters           Counters
```

Each journey keeps its own current state, entry time, transaction counters, lifetime counters, and last transaction date. A transition in one journey leaves the other's state alone.

## What the current window computes first

Classification writes temporary results before durable state moves:

```text
Current Window
     │
     ▼
Classification
     │
     ▼
Temporary Results
     │
     ├───────────────► State Snapshot
     │
     └───────────────► State Update
```

The temporary set holds the winning rule per user, journey, and transaction rank, plus whatever the state update needs later. Event generation stays out of it: one rule can emit many downstream events, but state updates read the transaction-level results so a single classification never counts twice through its many events.

## State snapshot

The snapshot runs after classification, right before the durable update. The pipeline lists every `(user, journey)` pair the window will modify, then per pair:

```text
Does a current state row exist?
          │
     ┌────┴────┐
     │         │
    Yes        No
     │         │
     ▼         ▼
Copy old     Record
state        "no previous row"
```

It saves enough to rebuild the old state:

```text
current state
state entry time
transaction count in state
last transaction date
first transaction date
lifetime transaction count
lifetime transaction amount
```

A flag notes whether the row existed at all. On restore, an existing row gets its saved values back while a row the failed window created gets deleted.

## Why the snapshot is an undo log

The snapshot answers one question:

> What did the state look like immediately before this window changed it?

```text
Before update
     │
     ├──► Snapshot
     │
     ▼
Current State
     │
     ▼
State Update
```

A good update makes the new state authoritative. A bad one leaves the snapshot to rebuild from.

## Validating the snapshot

Snapshots get checked, not trusted. The pipeline writes the expected pairs, then counts snapshot rows against distinct `(user, journey)` pairs under classification:

```text
Pairs to update
      │
      ▼
Create snapshot
      │
      ▼
Compare counts
   ┌──┴───┐
   │      │
 Match   Mismatch
   │      │
   ▼      ▼
Proceed  Fail
```

An incomplete snapshot stops the update. The pipeline never enters a state where some users restore and others cannot.

## State update

Once the snapshot passes, classification results aggregate by `(user, journey)`. A window can hold several transactions per user, but the table takes one consolidated update each:

```text
Transaction 1 ─┐
Transaction 2 ─┼─► Aggregate Window State ─► State Table
Transaction 3 ─┘
```

### Current state

The highest transaction rank in the window decides.

### Last transaction date

The latest grouped date becomes the user's latest.

### First transaction date

An existing value survives. The window fills it in only when nothing exists yet.

### Lifetime transaction count

```text
new lifetime count
=
existing lifetime count
+
transactions classified in this window
```

### Lifetime transaction amount

```text
new lifetime amount
=
existing lifetime amount
+
amount classified in this window
```

### Count inside the current state

The in-state counter continues from its old value, resets on a state change, or follows a configured preservation rule when a transition explicitly keeps the anchor. The rule engine models real state machines instead of treating windows as independent.

## Why state update cannot simply be retried blindly

Start from:

```text
lifetime transaction count = 100
```

Five qualifying transactions arrive, and the update writes:

```text
100 + 5 = 105
```

Run the same update against the already updated state:

```text
105 + 5 = 110
```

Five transactions counted twice. Amounts break the same way, so a state update is not inherently idempotent, and the pipeline restores old state before retrying.

## Crash during state update

```text
1. CLASSIFY          ✓
2. STATE_SNAPSHOT    ✓
3. STATE_UPDATE      ✗
4. PUBLISH           -
```

The snapshot already holds pre-update state. Recovery restores it, resets the touched stages, re-runs classification, takes a fresh snapshot, and applies the update exactly once. The cumulative update never runs again over partially written state.

## Restore behavior

Two cases, two paths.

### Existing state row

```text
Saved State
    │
    ▼
Restore
    │
    ▼
Previous values
```

Saved values overwrite the modified row.

### Newly created state row

```text
No previous state
       │
       ▼
Window creates row
       │
       ▼
Window fails
       │
       ▼
Restore
       │
       ▼
Delete created row
```

That is why the snapshot records whether each row predated the window.

## Why the restore is safe to repeat

Restore is idempotent by construction. Rewriting saved values changes nothing the second time, and deleting an already deleted row harms nothing:

```text
Restore
Restore again
Restore again
```

Every pass lands on the same previous state, which keeps operational recovery tooling simple.

## Recovery on the next pipeline run

A window with a good snapshot but a bad update gets caught by the next run, which restores first and re-attempts classification:

```text
CLASSIFY
STATE_SNAPSHOT
STATE_UPDATE
```

all marked as still to do. When the update already succeeded, restoration does nothing, so a later failure like a publish crash never rolls back committed state.

## Operator rewind

Operators sometimes rewind processed windows on purpose:

```text
Operator requests rewind
          │
          ▼
Acquire pipeline lock
          │
          ▼
Restore state from snapshots
          │
          ▼
Remove downstream artifacts for those windows
          │
          ▼
Remove processing ledger entries
          │
          ▼
Replay from the selected point
```

The same snapshot mechanism serves. Rewinds take the pipeline lock because they touch the same user state as normal classification. Internal API paths, lock names, command payloads, and environment specifics stay out of this write-up.

## Snapshot retention

Preimages live for a bounded period:

```text
Recent window
     ↓
Rollback available

Older window
     ↓
Preimage eventually expires
     ↓
Automatic restoration no longer possible
```

Retention is operations configuration, not state machine behavior, so no period is named here. Recovery work has to respect that boundary.

## Why two windows cannot run together

State update reads current state and adds the window's counts. Classification reads the same current state to pick matching rules. Safe ordering looks like:

```text
Window N
   │
   ▼
Read State
   │
   ▼
Update State
   │
   ▼
Commit
   │
   ▼
Window N+1
```

Two concurrent windows could read identical starting values and both write from them, losing updates, doubling increments, or landing wrong transitions. Four controls stop that: Kubernetes scheduling against overlapping pods, a database lock around the pipeline, ledger detection of active runs, and oldest-first execution inside one process. Operator rewinds take the same lock.

## Why state updates are aggregated

The state table tracks `(user, journey)`, not events. Three transactions in one window:

```text
Transaction A
Transaction B
Transaction C
```

aggregate first:

```text
A + B + C
    │
    ▼
One user-journey state update
```

Event-level classification stays apart from user-level state, and one transaction never counts extra times through its many downstream events.

## The deeper design pattern

A small state machine with explicit recovery points:

```text
             Previous State
                   │
                   ▼
              Classification
                   │
                   ▼
            Temporary Results
                   │
                   ▼
              State Snapshot
                   │
                   ▼
              State Update
                   │
                   ▼
              New State
```

Failure past the snapshot:

```text
New State
   │
   X failure
   │
   ▼
Restore Previous State
   │
   ▼
Retry
```

This shape fits anywhere processing runs in windows, output leans on previous state, updates accumulate, retries happen, and downstream correctness needs ordered state.

## Design trade-offs

### 1. Statefulness versus simplicity

A stateless classifier would be simpler, but it could not read previous state, lifetime counters, or time in state.

### 2. Snapshot storage versus recovery

Preimages cost storage and retention work, and buy a real recovery boundary for cumulative updates.

### 3. Sequential execution versus throughput

Windows run one after another, trading parallelism for temporal ordering of the state machine.

### 4. Cumulative updates versus idempotency

Cumulative counters serve stateful classification, and their updates need explicit rollback or transactional protection.

## A useful mental model

The state pipeline works like a database transaction stretched across a workflow:

```text
               WINDOW
                  │
                  ▼
          Build Classification
                  │
                  ▼
           Snapshot State
                  │
                  ▼
            Update State
             ┌────┴────┐
             │         │
          Success    Failure
             │         │
             ▼         ▼
         New State   Restore
                       │
                       ▼
                     Retry
```

Each operation declares the state it needs first and how to rebuild the old state when it fails, instead of relying on blind retries.

## Closing thought

Windowed processing gets hard exactly where current output depends on earlier state:

```text
What state does the current window depend on?
        ↓
Which users will the window modify?
        ↓
How is the previous state protected?
        ↓
What happens if the update fails halfway?
        ↓
Can the update be retried safely?
        ↓
How is window ordering preserved?
```

State snapshot plus controlled update plus rollback plus sequential windows turns a fragile cumulative write into a recoverable transition system. State handling belongs in the correctness model of a windowed pipeline, not in its implementation details.

*Next: enrichment, vendor routing, delivery, and the DLQ. Till then enjoy your life and happy engineering!*
