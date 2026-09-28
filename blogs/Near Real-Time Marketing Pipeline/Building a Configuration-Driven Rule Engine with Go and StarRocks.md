---
title: Building a Configuration-Driven Rule Engine with Go and StarRocks
slug: configuration-driven-rule-engine
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - go
  - starrocks
  - architecture
  - real-time
description: How classification rules live as configuration rows instead of application code, and what keeps that flexibility safe.
reading_time: 16
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 5
---
Production classification does not have to compile business rules into the application binary. Our rules live as configuration data. Each processing window syncs the active rules into the analytical engine, and the classification SQL itself never changes:
```text
Rule Management
      │
      ▼
Configuration Store
      │
      ▼
Configuration Sync
      │
      ▼
Analytical Rule Copy
      │
      ▼
Static Classification SQL
      │
      ▼
Classified Users
```
Business rule edits ship without rebuilding or redeploying the pipeline.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=IAWoC2XS0q3aJvCfNxR-">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=IAWoC2XS0q3aJvCfNxR-&type=embed" /></a>

## The path

```text
Relational Configuration Store
          │
          │ Admin API
          ▼
   Configuration Sync
          │
          ▼
 Analytical Rule Copy
          │
          ▼
 Classification SQL
          │
          ▼
 Temporary Classification Results
```

The relational store is the editable source of truth. The analytical engine holds a copy the classification queries read. The SQL stays fixed while the data feeding it changes with the configured rules.

## Why store rules as rows

The usual way to build a rule system is to write every rule into application logic:

```text
if condition A:
    classification A
elif condition B:
    classification B
...
```

That holds up while rules rarely change. It gets expensive once product or operations teams start editing conditions often.

Here a rule is data. A rule row can carry fields like:

```text
from_state
to_state
events_to_fire
amount bounds
gap-day constraints
lifetime constraints
vertical constraints
source constraints
transaction-type constraints
priority
catch-all flag
```

The exact schema stays out of this write-up. The classification query never learns the business meaning of any single rule. It reads the configured columns and applies whatever each row says:

```text
Application Code
      │
      │ stable
      ▼
Classification Engine

Configuration Data
      │
      │ changes frequently
      ▼
Classification Rules
```

Processing code stays put while classifier behavior moves.

## Adding or modifying a rule

Rules change through the configuration API:

```text
Admin / Frontend
      │
      ▼
API Service
      │
      ▼
Configuration Database
      │
      ▼
Next pipeline window
      │
      ▼
Configuration Sync
      │
      ▼
Analytical Rule Copy
```

No redeploy, and the classification SQL files stay untouched. Two kinds of change stay apart:

```text
Business-rule change
        ↓
Configuration update

Processing-logic change
        ↓
Application / SQL change
```

## Validating rules before they reach production processing

A rule engine that accepts anything becomes dangerous fast, so the API validates before storing. The main check is totality: for every journey with enabled rules, one enabled catch-all rule has to cover the leftovers:

```text
Journey
  │
  ├── Specific Rule A
  ├── Specific Rule B
  ├── Specific Rule C
  └── Catch-All Rule
```

Without a catch-all, a transaction matching nothing narrow falls through unclassified.

The configuration layer also checks rules that preserve a previous classification anchor. The standing policy:

> Configuration should be validated at write time rather than allowing invalid rule sets to reach the processing engine.

## Configuration synchronization

The orchestrator syncs configuration before deduplication and classification:

```text
1. Read enabled rules
2. Validate that rules exist
3. Copy the complete rule set
4. Verify the analytical copy
5. Start classification
```

The copy goes in whole, never row by row. Classification then sees either the previous rule set or the new complete one:

```text
Previous rule set
        OR
New complete rule set
```

never a mix like:

```text
Rule A from old set
Rule B from new set
Rule C from old set
...
```

A half synced configuration must never reach classification. And when the store holds no active rules at all, the window fails closed instead of running an empty classifier.

## Why copy the rules into the analytical engine

The configuration database manages configuration. The analytical engine scans the transaction window and joins it against user state and rules. So classification runs roughly like:

```text
Transactions
     │
     ├──────────────┐
     ▼              ▼
User State      Rule Copy
     │              │
     └──────┬───────┘
            ▼
       Classification
```

Joining the rule copy on the analytical cluster keeps it beside the big transaction and state scans. The API database remains the source of truth for editing, authentication, and audit needs.

## Why the classification SQL stays static

The orchestrator never generates a fresh SQL program per rule change. Classification statements stay stable and take runtime parameters:

```text
window start
window end
processing phase
maximum processing phases
```

Rules stay rows in a table:

```text
Runtime parameters
        ↓
SQL execution context

Classification rules
        ↓
Rows joined by the SQL
```

Rule text never turns into executable application code, which keeps the configuration layer from growing into a code generator.

## One winning rule per journey

The classifier can run several journeys over one transaction:

```text
Transaction
    │
    ├────► Overall Journey
    │
    └────► Vertical-specific Journey
```

One transaction can join multiple independent journeys. Inside a single journey, selection stays exclusive. When several rules match, the system ranks them and picks exactly one winner:

```text
Transaction + Journey
        │
        ▼
Matching Rules
   │    │    │
   ▼    ▼    ▼
 R1    R2    R3
   \    │    /
    \   │   /
     priority
        │
        ▼
   Winning Rule
```

Ordering is deterministic: lower priority wins, and equal priorities break on the lower rule identifier. The same transaction classifies identically no matter what order the database returns matching rows in.

## Catch-all rules

A catch-all is a configured rule like any other, not a hard-coded branch:

```text
Specific Rule A ── match ──► Winner

Specific Rule B ── no match
Specific Rule C ── no match
Catch-All       ───────────► Winner
```

It wins only when no higher priority specific rule matches, which keeps the fallback inside the same rule model as everything else.

## One rule can emit multiple events

A winning rule does not map to exactly one downstream event. One rule can name several:

```text
Winning Rule
      │
      ├──► Event A
      ├──► Event B
      └──► Event C
```

Enrichment expands the configured events into individual records:

```text
One rule
   ≠
One event
```

One classification decision intentionally produces many downstream events.

## Multi-phase classification

Some transactions need more than one classification pass, so later transaction ranks go through extra phases with the same selection mechanism. A maximum phase count guards against unbounded traversal, and past that limit the pipeline falls back in a controlled way instead of generating phases forever. The production limit stays out of this article.

## Rule consistency during a processing window

The rule set gets read once, at the window's start:

```text
Window starts
     │
     ▼
Config Sync
     │
     ▼
Rule Copy established
     │
     ▼
Dedup
     │
     ▼
Classification
```

Later stages read that analytical copy. When an administrator edits a rule mid classification:

```text
10:00  Config Sync
       ↓
       Rule Set A copied
       ↓
10:05  Admin changes configuration
       ↓
       RDS now contains Rule Set B
       ↓
10:10  Classification continues
       ↓
       Uses Rule Set A
```

The running window keeps Rule Set A. The next window syncs again and picks up Rule Set B. Every window gets a stable rule set with no separate rule-versioning system behind it.

## Retry behavior

A failed window does not always sync configuration again. When the sync stage already passed, the retry resumes from the later failed stage on the same analytical rule copy:

```text
CONFIG_SYNC     ✓
DEDUP           ✓
CLASSIFY        ✗
STATE_SNAPSHOT  -
STATE_UPDATE    -
PUBLISH         -
```

Only a failed sync re-runs the sync, against whatever configuration is active then. Retrying a window and starting a new window are different operations: a retry does not automatically mean reading configuration again.

## What is actually frozen

No permanent rule-version table tracks every window. The working rule set is simply the copy in the analytical rule table after a good sync. Rule freezing and user-state protection stay separate:

```text
Rule Copy
    ≠
User State Backup
```

The user-state snapshot guards classification state. The synced rule copy steadies the current window's rules.

## Configuration-driven design versus code-driven design

Code-driven:

```text
Business Rule
     │
     ▼
Source Code
     │
     ▼
Build
     │
     ▼
Deploy
     │
     ▼
New behavior
```

Configuration-driven:

```text
Business Rule
     │
     ▼
Configuration Store
     │
     ▼
Configuration Sync
     │
     ▼
New behavior
```

The second model drops deployment from rule changes. In exchange the configuration layer joins the correctness boundary, which means it needs validation, access control, auditability, safe synchronization, and predictable fallbacks.

## Design trade-offs

### 1. Flexible rules versus query complexity

Conditions in configuration buy flexibility, and the classification SQL grows more sophisticated to evaluate generic rows.

### 2. Runtime flexibility versus configuration safety

Rules change without deploys, so bad configurations get rejected before the processing path.

### 3. Analytical copy versus operational source of truth

The relational database suits editing and administration. The analytical copy suits big joins and scans.

### 4. Stable processing code versus richer configuration

The application stays stable only while the rule schema can say what the business needs. A condition outside the model still forces code or schema work.

### 5. Window consistency versus immediate rule propagation

A rule edit never touches a running window. Processing stays deterministic, and the change lands at the next sync boundary instead of instantly.

## A useful mental model

Two planes make up the system:

```text
              CONTROL PLANE
        ┌──────────────────────┐
        │ Frontend / Admin API │
        │ Rule validation      │
        │ Configuration Store  │
        └──────────┬───────────┘
                   │
                   ▼
             Config Sync


              DATA PLANE
        ┌──────────────────────┐
        │ Transaction data     │
        │ User state            │
        │ Rule copy             │
        │ Classification SQL   │
        └──────────┬───────────┘
                   │
                   ▼
             Classified Output
```

The control plane decides what the rules are. The data plane runs those rules over the current window.

## Closing thought

A configuration-driven rule engine goes further than moving `if/else` statements into a database. The work is drawing a safe boundary across business configuration, validation, synchronization, deterministic execution, state changes, and downstream events:

```text
Business configuration
        ↓
Validation
        ↓
Synchronization
        ↓
Deterministic execution
        ↓
State changes
        ↓
Downstream events
```

With that boundary explicit, the application holds still while business behavior moves through configuration. Keeping it safe means every change stays valid, every window sees one consistent rule set, and stateful effects stay recoverable.

*Next: enrichment, vendor routing, and delivery. Till then enjoy your life and happy engineering!*
