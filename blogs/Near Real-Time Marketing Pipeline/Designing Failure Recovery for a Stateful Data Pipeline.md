---
title: Designing Failure Recovery for a Stateful Data Pipeline
slug: failure-recovery-stateful-pipeline
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - starrocks
  - recovery
  - reliability
  - real-time
description: How the pipeline recovers from ambiguous state updates: snapshots, restore paths, stale-run detection, and ordered window recovery.
reading_time: 18
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 7
---

A stateful classification pipeline recovers differently than a stateless batch job. The current window rewrites durable user state with cumulative changes. When an update may have partially or fully landed, running it again can corrupt the totals. Recovery sorts every failure into one of three buckets:

```text
Failure before state protection
        ↓
Retry normally

Failure after state protection but before state-update success
        ↓
Restore previous state
        ↓
Rebuild classification
        ↓
Apply state update once

Failure after state update succeeds
        ↓
Do not automatically roll back state
```

The execution ledger decides which bucket each failure lands in.

## The recovery path

Snapshot succeeded while the state update did not:

```text
State Snapshot = SUCCESS
State Update   != SUCCESS
        │
        ▼
Restore previous state
        │
        ▼
Reset classification-related stages
        │
        ▼
Run classification again
        │
        ▼
Take a new snapshot
        │
        ▼
Apply state update once
```

Later windows stay blocked until the failed one completes, which keeps the state machine's temporal ordering intact.

## Why a simple retry is not enough

Start from a lifetime transaction count of:

```text
100
```

Five transactions arrive, and the update writes:

```text
100 + 5 = 105
```

Now the process dies after the database applied the update but before the pipeline recorded success. A blind retry runs:

```text
105 + 5 = 110
```

Five transactions counted twice. The trouble is not that the job failed. The trouble reads:

> We no longer know whether the cumulative state mutation happened.

That unknown is exactly what the pre-update snapshot exists to resolve.

## Where recovery starts

Recovery runs before window processing whenever the ledger shows a finished snapshot with an unfinished state update:

```text
Snapshot completed
        AND
State update not completed
```

### Failure before snapshot

Classification dying before the snapshot leaves nothing to restore:

```text
Classification
      X
Snapshot not taken
State not updated
```

Classification simply runs again.

### Failure after state update

A successful state update with a failed publish keeps its state:

```text
State Update = SUCCESS
Publish = FAILED
```

Automatic recovery never rolls back committed state. Delivery recovery takes over from there instead.

## Failure boundaries inside a window

Window stages run in order:

```text
Configuration Sync
        ↓
Deduplication
        ↓
Classification
        ↓
State Snapshot
        ↓
State Update
        ↓
Publish
```

A later wake skips stages that already passed. A failed stage retries under the configured attempt policy, and whatever stays incomplete waits for the next scheduled wake:

```text
Successful stage
      ↓
do not repeat unnecessarily

Failed stage
      ↓
retry

State-update ambiguity
      ↓
restore before retrying
```

## In-wake retries versus recovery

Two mechanisms, two moments.

### Retry inside the current execution

A stage that errors can run again in the same process. The retry covers the stage alone and restores nothing:

```text
Stage fails
   ↓
Retry stage
```

### Recovery on a later execution

A process that exits mid state-update leaves the ledger for the next run. Finding snapshot success with update incomplete, it restores pre-update state before rebuilding:

```text
Later execution
      ↓
Detect uncertain update
      ↓
Restore
      ↓
Reclassify
      ↓
Apply once
```

Failures from before the state table got touched never pay for restoration they do not need.

## What happens when the execution ledger update fails?

Pipeline state itself persists in a relational execution ledger. A step moves:

```text
PENDING
   ↓
RUNNING
   ↓
SUCCESS
```

or:

```text
PENDING
   ↓
RUNNING
   ↓
FAILED
```

Ledger writes belong inside the correctness boundary. A stage that succeeds without persisting its success stops instead of assuming the ledger kept up. That fail-closed choice beats a later run skipping work that never actually finished.

## Why the ledger is not the same as the data state

Two kinds of state move through this system:

```text
Processing state
    ↓
What stage has completed?

Business state
    ↓
What is the current state of the user?
```

The execution ledger tracks the first. The user-state table holds the second. Recovery joins them:

```text
Execution Ledger
       │
       │ detects ambiguous transition
       ▼
State Preimage
       │
       ▼
Restore Business State
       │
       ▼
Resume Processing
```

Deleting or editing a ledger entry restores no user state. The two stores answer different questions.

## Recovering a partial state update

The recovery case that matters most:

```text
Classification       ✓
State Snapshot       ✓
State Update         ?
Publish              -
```

The question mark asks whether the state update happened. The pipeline refuses to guess either way:

```text
Restore known-good state
        ↓
Reset affected stages
        ↓
Rebuild classification
        ↓
Take new snapshot
        ↓
Apply state update
```

Ambiguity converts into known state before any retry.

## How the restore works

The snapshot records per affected pair whether a state row predated the window. Two paths follow.

### Existing state

```text
Previous state existed
        ↓
Restore saved values
```

Saved values overwrite whatever the uncertain update left.

### Newly created state

```text
No previous state
       ↓
Failed window created state row
       ↓
Restore
       ↓
Delete created row
```

Restoration never writes nulls or synthetic values into a row the window invented.

## Why the restore itself is retry-safe

Recovery repeats safely. Restoring an existing row rewrites identical saved values:

```text
Restore(value X)
Restore(value X)
Restore(value X)
```

still leaves:

```text
value X
```

Deleting a created row twice harms nothing once it is gone. Restoration converges on the old state no matter how often it runs, even though the cumulative update underneath never could.

## Rebuilding classification after restore

Restored state means rebuilt classification. Temporary data from the failed attempt gets discarded rather than trusted:

```text
Restore
  ↓
Clear temporary classification
  ↓
Read restored state
  ↓
Re-run classification
  ↓
Generate fresh state transition
```

The failed attempt may have derived intermediates from uncertain state. Rebuilding dependent work beats patching partial results.

## Why publish failure does not trigger state rollback

```text
Classification     ✓
Snapshot           ✓
State Update       ✓
Publish            X
```

User state already reads correct. Rolling it back over a Kafka failure would tangle two separate concerns:

```text
Business state
      ≠
Event delivery
```

Recovery retries publish with state untouched. Publish retries never enter the state-restore path.

## Stale or abandoned executions

A process can vanish while its ledger row still reads:

```text
RUNNING
```

```text
Pod starts
   ↓
Stage runs
   ↓
Pod crashes
   ↓
Ledger still says RUNNING
```

The next run separates currently active from stale:

```text
Currently active
```

```text
Stale execution
```

Heartbeat data plus a staleness threshold does the sorting. Stale executions get cleaned up, abandoned queries die, and the run marks failed so normal recovery can proceed. Timeout values and cleanup identifiers stay implementation detail.

## Why stale detection matters

Without it:

```text
Dead process
    ↓
RUNNING forever
    ↓
Every future scheduler wake sees overlap
    ↓
Pipeline remains blocked
```

With it:

```text
Dead process
    ↓
RUNNING becomes stale
    ↓
Mark FAILED
    ↓
Recovery becomes possible
```

Process death turns from a permanent block into a recoverable condition.

## Why later windows remain blocked

```text
Window A → failed during state update
Window B → ready
Window C → ready
```

Running B and C around A looks tempting:

```text
B → C
```

while A sits unresolved. B's classification reads the user state A was supposed to leave, so processing against uncertain state produces wrong transitions. Safe ordering recovers first:

```text
A → recover
  ↓
B
  ↓
C
```

Incomplete windows process oldest first and stop at the first failure.

## Failure matrix

| Failure point | Business state | Recovery |
|---|---|---|
| Configuration sync | Unchanged | Retry failed stage |
| Deduplication | Unchanged | Retry failed stage |
| Classification | Unchanged | Rebuild classification |
| State snapshot | Unchanged | Retry snapshot |
| Snapshot succeeded, state update uncertain | Potentially changed | Restore pre-update state, then rebuild |
| State update succeeded, publish failed | Correct | Retry publish only |
| Process becomes stale during a stage | Depends on stage | Mark stale execution failed, then apply normal recovery rules |

The state update draws the line. Before it, plain stage retries do. During it, an undo image is required. After it succeeds, downstream failures never roll state back.

## Idempotency map

Stages retry differently:

| Stage | General retry characteristic |
|---|---|
| Configuration synchronization | Replacing the analytical configuration copy is repeatable |
| Deduplication | Historical identity checks prevent accepted duplicates |
| Classification | Temporary results can be rebuilt |
| State snapshot | Existing snapshot for the same window can be replaced |
| State update | Cumulative update is not safe to repeat without restoration |
| Restore | Designed to converge on the previous state |
| Publish | At-least-once; a replay can occur after Kafka acknowledgement but before checkpoint persistence |

"Make the whole pipeline idempotent" states the goal too loosely. Each stage needs its own recovery strategy.

## Operator rewind

Automatic recovery fixes uncertain updates. Operators sometimes need the opposite direction on purpose:

> What if we intentionally need to move the system back to an earlier successful point?

That is a rewind, not failure recovery:

```text
Choose rewind point
       ↓
Acquire pipeline lock
       ↓
Restore affected state from preimages
       ↓
Remove downstream artifacts for rewound windows
       ↓
Reset execution history
       ↓
Replay forward
```

The same preimages serve both uses. API paths, payloads, lock names, and command details stay out of this write-up.

## Why rewind needs the same lock

Rewinds rewrite the user state normal classification updates:

```text
Normal state update
        +
Rollback
```

against the same rows. Mixed together they leave half old, half new state. Normal processing and operator rewinds hold one shared execution boundary so restoration never races a live update.

## What happens after a successful state update

A good update becomes the new truth. Its preimage lives only as long as operations needs it:

```text
Old preimage
      ↓
expired
      ↓
automatic restoration unavailable
```

Rewind reach ends where retention ends. That bound is a design trade, not an accident.

## Failure recovery as a state machine

```text
                  ┌───────────────┐
                  │   Processing  │
                  │    Window     │
                  └───────┬───────┘
                          │
                    Classification
                          │
                          ▼
                   State Snapshot
                          │
                          ▼
                    State Update
                          │
              ┌───────────┴───────────┐
              │                       │
          Successful               Uncertain
              │                       │
              ▼                       ▼
           Publish                 Restore
              │                       │
              ▼                       ▼
          Complete               Reclassify
                                      │
                                      ▼
                                  New Snapshot
                                      │
                                      ▼
                                  New Update
```

The pipeline transitions back to known state before repeating a non-idempotent operation. It never just retries commands.

## Design trade-offs

### 1. Snapshot storage versus recovery capability

Preimages cost storage and cleanup work, and make uncertain cumulative updates recoverable.

### 2. Sequential windows versus throughput

Newer windows wait behind earlier failures, trading parallelism for temporal consistency.

### 3. Fail-closed ledger writes versus availability

Halting on unwritable ledger state costs availability and stops the system from assuming finished work.

### 4. Bounded preimage retention versus operational rewind

Longer retention widens the recovery horizon and eats more storage.

### 5. Rebuild versus patch

Rebuilding classification after restore repeats analytical work instead of patching intermediates built on uncertain state.

## A useful mental model

Recovery keeps a known-good checkpoint:

```text
Known-good state
      │
      ▼
Snapshot
      │
      ▼
State transition
      │
      ├── Success ──► New known-good state
      │
      └── Failure ──► Restore previous checkpoint
```

The snapshot draws the line across which ambiguity stays recoverable.

## Closing thought

The hardest failure in a stateful pipeline never announces itself as an error. It looks like an open question:

```text
Did the state update happen?
        │
        ├── Yes
        └── No
```

When neither answer is safe, the system restores known-good state, rebuilds the dependent work, applies the transition once, and continues delivery. That discipline travels beyond classification pipelines. Any workflow with cumulative, non-idempotent state changes recovers on the same five pieces: a known-good pre-state, explicit execution progress, clear failure boundaries, controlled concurrency, and a safe way to reconstruct the transition.

Recovery removes ambiguity before retrying a non-idempotent operation.

*Next: enrichment, vendor routing, and delivery. Till then enjoy your life and happy engineering!*
