---
title: Separating Classification from Vendor Enrichment
slug: separating-classification-vendor-enrichment
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - architecture
  - enrichment
  - real-time
description: Why classification emits a canonical event and lets enrichment, routing, and delivery workers handle everything vendor-specific.
reading_time: 16
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 9
---
Classification should decide what happened to a user. It should not also decide how every downstream platform wants that event delivered:
```text
Classification
     │
     ▼
Canonical Classified Event
     │
     ▼
Enrichment
     │
     ▼
Vendor-specific Routing
     │
     ▼
Vendor-specific Payload
     │
     ▼
External API
```
The classification pipeline stays clear of vendor identifiers, routing rules, payload schemas, and delivery behavior.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=ul05sS2aqHAgHuH3L1fN">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=ul05sS2aqHAgHuH3L1fN&type=embed" /></a>

## The path

```text
Analytical Classification
          │
          ▼
   Canonical Event Topic
          │
          ▼
    Enrichment Service
          │
          ├── Lookup user/device identifiers
          │
          └── Resolve routing configuration
          │
          ▼
 Vendor-specific Kafka Topics
          │
          ▼
    Vendor Delivery Workers
          │
          ▼
     External HTTP APIs
```

Classification produces an event with one canonical internal shape. Everything vendor-specific joins later.

## What classification should know

Classification already sees what it needs: transaction information, current user state, configured rules, and the PII attributes the logic requires. It has no business knowing:

- vendor-specific user identifiers;
- vendor-specific event names;
- which Kafka topic a vendor reads;
- the exact JSON a vendor expects; or
- the external API endpoint for delivery.

```text
                    Classification
                         │
                "User belongs to X"
                         │
                         ▼
                  Canonical Event
```

The canonical event carries the business decision with no downstream platform baked in.

## Why vendor identifiers are added later

External platforms rarely agree on identity. One internal user maps outward like:

```text
Internal User ID
      │
      ├────► Vendor A User ID
      ├────► Vendor B User ID
      └────► Vendor C User ID
```

Stuffing every vendor identifier into the classification table would grow its schema with each new integration. The enrichment service resolves identifiers after classification instead, and the internal model never counts downstream integrations.

## Why enrichment is a separate service

Classification runs windowed and stateful. Enrichment runs continuously with its own failure shape:

```text
Classification Window
        │
        ▼
Canonical Event
        │
        ▼
Enrichment Service
        │
        ▼
External Identifier Lookup
```

When the identifier store goes down mid-flight, a combined design stalls everything:

```text
Identifier Store Down
        ↓
Classification blocked
        ↓
State update delayed
```

Split apart, classification finishes, the canonical event waits, and enrichment retries on its own:

```text
Classification completes
        ↓
Canonical event remains available
        ↓
Enrichment retries independently
```

An identifier outage never forces already committed classification state to roll back.

## What the enrichment service actually does

The enrichment layer reads the canonical event and looks up the external identifiers downstream platforms need:

```text
Canonical Event
      │
      ▼
Lookup internal user
      │
      ▼
External identifiers
      │
      ▼
Enriched Event
```

Lookups hit a dedicated low-latency store. One user lookup serves every downstream platform on the event, so a single fetch covers several destinations at once.

## Handling a missing identifier

A missing mapping does not make the event itself invalid. Enrichment emits the event with the optional identifier absent instead of dropping the whole classification:

```text
Missing vendor identifier
        ≠
Invalid classification
```

Delivery then decides per event whether a gap means acceptable, retryable for now, or permanently invalid. Enrichment resolves data. Delivery sets policy.

## Failure isolation with a circuit breaker

External lookups add dependency risk, and a breaker keeps the enrichment service from hammering a failing dependency:

```text
             Lookup Dependency
                    │
          ┌─────────┴─────────┐
          │                   │
       Healthy             Failing
          │                   │
          ▼                   ▼
       Lookup             Open breaker
          │                   │
          ▼                   ▼
    Continue flow       Degrade / retry later
```

The breaker earns its keep on big batches: with many users and a dead dependency, the service flips to its degraded path instead of firing off thousands of doomed requests.

## Concurrency control

Lookups for a batch with many distinct users can choke the dependency, so enrichment caps concurrent lookups instead of fanning out without bound:

```text
1000 users
   │
   ▼
Bounded lookup concurrency
   │
   ├── Lookup
   ├── Lookup
   ├── Lookup
   └── ...
```

The cap guards both sides. Its exact value is an implementation detail and stays out of this write-up.

## Why the router does not build vendor JSON

The enrichment and router layer emits an enriched canonical event. Full vendor request schemas belong to the delivery worker:

```text
Enrichment
     │
     ▼
Enriched Event
     │
     ▼
Vendor Topic
     │
     ▼
Vendor Worker
     │
     ▼
Vendor JSON
```

Two questions, two owners. The router answers where an event goes and which identifiers it needs:

> Where should this event go and what identifiers does it need?

The vendor worker answers how the destination API wants it shaped:

> How should this event be represented for the destination API?

## Configuration-driven routing

Routing reads configuration. A routing rule conceptually names:

```text
Internal Event Type
        +
Target Vendor
        +
Target Topic
        +
Vendor Event Name
        +
OS / platform condition
```

The router holds the active configuration in memory and maps each event to its destinations. Routing edits never touch classification SQL.

## One event can have multiple destinations

A canonical event can match several routing rules at once:

```text
                    Canonical Event
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
           Vendor A    Vendor B    Vendor C
            Topic       Topic       Topic
```

Every destination gets its own enriched copy, and classification never learns how many integrations exist.

## Why vendor topics are separate

Each vendor gets its own Kafka topic:

```text
                 Canonical Event
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Vendor A      Vendor B      Vendor C
       Topic         Topic         Topic
          │            │            │
          ▼            ▼            ▼
       Worker A     Worker B     Worker C
```

When Vendor A goes down, its worker retries while Vendors B and C keep consuming:

```text
Vendor A Worker
      ↓
Retry / Failure
```

One outage never blocks unrelated destinations.

## Configuration-driven event naming

Internal event names need not match external ones:

```text
Internal Event
      │
      ▼
Routing Configuration
      │
      ├──► Vendor A Event Name
      ├──► Vendor B Event Name
      └──► Vendor C Event Name
```

Different platforms name the same business event differently. Classification emits the internal semantic event, and routing translates it per destination.

## OS-specific routing

Routing configuration can carry platform constraints:

```text
Event
 │
 ├── iOS       → Vendor A
 │
 ├── Android   → Vendor B
 │
 └── Any OS    → Vendor C
```

The router normalizes platform information before matching. Normalization internals stay out of this article.

## Where the vendor JSON is built

The vendor worker takes the enriched event and builds the destination HTTP body from configuration:

```text
Enriched Event
      +
Vendor Template
      ↓
Vendor-specific JSON
```

A template field names a destination JSON path, a source event field, a default, and a transformation. One canonical event becomes many destination payloads while upstream classification never learns those schemas.

## Adding a new vendor

New integrations land without touching the classification engine:

```text
Existing Platform
      │
      ├── Classification SQL
      ├── Enrichment Service
      └── Vendor Worker

New Vendor
      │
      ├── Routing configuration
      ├── Vendor topic
      ├── Payload configuration
      └── Worker deployment
```

That pays off more with every destination added. Deployment and configuration mechanics stay implementation-specific.

## Why configuration-driven payload generation matters

Without it, onboarding a vendor means:

```text
New vendor
    ↓
Change classification code
    ↓
Change enrichment code
    ↓
Change payload builder
    ↓
Redeploy everything
```

With the split:

```text
New vendor
    ↓
Add routing configuration
    ↓
Add payload configuration
    ↓
Deploy isolated worker
```

The core platform holds still. The price is a configuration model expressive enough to describe the destination API.

## Handling enrichment failures

Enrichment ends one of three ways:

```text
Lookup succeeds
      ↓
Publish enriched event

Lookup says mapping does not exist
      ↓
Publish event with missing optional identifiers

Lookup dependency fails
      ↓
Do not acknowledge the batch
      ↓
Retry
```

"Mapping does not exist" and "dependency failed" are different facts. The first can be legitimate data. The second is infrastructure that may heal on retry. Merging them either retries pointlessly or drops events quietly.

## Why the consumer commits manually

The enrichment service handles offsets explicitly so it decides when a batch counts as done:

```text
Consume
  ↓
Enrich
  ↓
Publish results
  ↓
Commit offset
```

A batch that fails before publishing keeps its offset uncommitted and runs again. The boundary stays clean.

## Preventing duplicate downstream publishes during a batch retry

Late batch failures can leave earlier events already published, so a full retry would duplicate them. The enrichment service tracks what it already accepted during the current attempt and skips re-emitting those items on an immediate in-process retry. That trims local retry noise. Durable duplicate handling still lives with event identity and destination delivery.

## What the split keeps apart

### Classification versus enrichment

```text
Classification
      │
      │ no external identifier dependency
      ▼
Canonical Event

Enrichment
      │
      │ external identifier dependency
      ▼
Enriched Event
```

A lookup failure never forces the classification engine to redo finished state transitions.

### Enrichment versus delivery

```text
Enrichment
     ↓
Vendor Topic
     ↓
Delivery Worker
```

Enrichment never builds vendor HTTP bodies.

### Vendor A versus Vendor B

```text
Vendor A topic ──► Worker A
Vendor B topic ──► Worker B
```

One destination's trouble stays its own.

## Why this architecture scales operationally

Each layer changes for its own reason:

```text
Classification changes
        ↓
Business rules / state logic

Enrichment changes
        ↓
User or device identifier mappings

Routing changes
        ↓
Destination selection

Payload changes
        ↓
External API schema

Delivery changes
        ↓
HTTP/retry behavior
```

Mixed together, any downstream edit can drag the whole pipeline along. Kept apart, each concern moves on its own.

## Design trade-offs

### 1. More services, more operational complexity

Classification, enrichment, routing, and delivery apart means more components, traded for stronger isolation and independent scaling.

### 2. Additional Kafka hops

Intermediate topics add network and serialization overhead, and buy buffering, decoupling, replayability, and per-vendor isolation.

### 3. Configuration flexibility versus validation complexity

Configuration-driven routing and payloads cut code changes, and the configuration system joins the correctness boundary.

### 4. Canonical model versus vendor-specific richness

A canonical event avoids vendor coupling, but it has to carry enough for enrichment and payload building downstream. A new integration needing data outside the canonical event or lookup layer still forces the upstream model to grow.

## A useful mental model

Four layers, four questions:

```text
┌─────────────────────────────┐
│  1. Classification          │
│  "What happened?"           │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  2. Enrichment              │
│  "What identifiers?"        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  3. Routing                 │
│  "Where should it go?"      │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  4. Delivery                │
│  "How should it look?"      │
└─────────────────────────────┘
```

## Closing thought

Event pipelines get hard to maintain the moment the classification engine learns about every downstream integration. The healthier boundary runs business decision, canonical event, enrichment, routing, vendor representation, delivery:

```text
Business decision
      ↓
Canonical event
      ↓
Enrichment
      ↓
Routing
      ↓
Vendor-specific representation
      ↓
Delivery
```

New or changed integrations leave the business engine alone. Business semantics stay independent from integration semantics: classification describes what happened, and everything downstream decides how that fact gets enriched, routed, shaped, and delivered.

*That closes the enrichment arc. Till then enjoy your life and happy engineering!*
