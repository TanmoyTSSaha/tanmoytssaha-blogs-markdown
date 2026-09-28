---
title: Configuration-Driven Vendor Onboarding
slug: configuration-driven-vendor-onboarding
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kafka
  - architecture
  - integrations
  - real-time
description: How new vendor integrations land as configuration plus an isolated worker, without touching the classification pipeline.
reading_time: 18
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 11
---
New external integrations should land without touching the core classification pipeline. Vendor behavior lives in configuration behind a reusable delivery worker:
```text
Classification
      │
      ▼
Canonical Event
      │
      ▼
Enrichment / Routing
      │
      ▼
Vendor-specific Kafka Topic
      │
      ▼
Generic Delivery Worker
      │
      ▼
External API
```
One application image serves many destinations. A runtime vendor identifier picks the configuration each worker runs with, keeping classification, enrichment, and publishing clear of any single platform's URL, auth style, batching habits, or JSON shape.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=aS7TeNgXEkjc-p2YySy0">View on Eraser<br /><img src="https://raw.githubusercontent.com/TanmoyTSSaha/tanmoytssaha-blogs-markdown/main/blogs/Near%20Real-Time%20Marketing%20Pipeline/assets/hld-11-onboarding.png" /></a>

## The configuration model

An administrative API manages vendor configuration in a relational database, split four ways:

Routing, delivery behavior, credentials, and payload fields stay apart. A new destination usually means configuration plus an isolated worker deployment, not core pipeline surgery.

## What is stored per vendor

A vendor configuration covers ideas like:

```text
vendor identifier
base URL
authentication method
batch strategy
rate limit
timeout
Kafka topic
consumer group
retry settings
unknown-platform behavior
```

Exact schema and field names stay out of this write-up. Secrets live in a separate credential store, and another configuration set describes the outbound JSON body.

## Routing rules

Routing answers one question:

> Which destination should receive this event?

```text
Internal Event Type
        +
Target Destination
        +
Event Name
        +
Platform Condition
        +
Target Topic
```

The router holds the active configuration in memory and refreshes it on a period:

```text
Routing configuration changes
            ↓
No classification deployment
            ↓
Router picks up the new rule
```

Classification never learns which destination wants an event.

## One event can fan out

A single canonical event can match several routing rules:

Each destination gets its own downstream representation, so one business event reaches many platforms while classification stays vendor-blind.

## Why routing is configuration-driven

Code-driven routing looks like:

```text
if event == A:
    send to vendor A
elif event == B:
    send to vendor B
...
```

Every integration costs a code change. Configuration-driven routing reads:

```text
Event
  ↓
Routing Rules
  ↓
Destination
```

Processing code holds still while routing data moves, which pays off most when integrations come and go often.

## Credentials are isolated

Vendor credentials sit apart from general delivery configuration:

The worker resolves whatever a destination needs at request time. Secrets stay encrypted, never plain text. Encryption key names, environment variables, credential keys, and admin API paths stay out of this article.

## Credential refresh

Credential changes never touch the core pipeline. The configuration service signals a refresh, the worker also refreshes on a period and after auth failures:

```text
Credential changed
      │
      ▼
Configuration Store
      │
      ▼
Worker refresh
      │
      ▼
Next HTTP request
```

Secret rotation stays separate from application logic.

## Building the vendor JSON

Payloads come from a field-oriented template model. A configuration row describes:

```text
Destination JSON path
Source event field
Mandatory / optional
Default value
Transformation
```

```text
source event field
        │
        ▼
transform
        │
        ▼
destination JSON path
```

No vendor gets a hard-coded payload builder of its own.

## Example of configuration-driven payload generation

A canonical event holds:

```text
user_id
email
phone
event_name
value
```

One destination wants:

```json
{
  "event": "...",
  "user": {
    "email": "..."
  }
}
```

another wants:

```json
{
  "name": "...",
  "user_id": "...",
  "value": 123
}
```

The same canonical event produces both shapes through mapping configuration. Classification never learns either JSON structure.

## Transformations

Template fields can transform values through normalization, hashing, or format conversion:

```text
normalization
hashing
format conversion
```

A field transforms before landing on its destination path, which covers external APIs with strict identifier rules. Internal transform registries and function names stay out of this write-up.

## Mandatory fields

Payload configuration can mark fields mandatory:

Validation runs before any HTTP request. An unresolvable mandatory field routes the event into the normal failure and DLQ flow instead of sending a broken external request.

## Stable JSON shape

Optional fields can still materialize when configuration demands a stable shape. Some external APIs expect consistent structure with empty values, and that behavior stays destination-specific configuration without implementation detail here.

## Previewing a payload

An admin preview endpoint renders a configured template against a sample event and returns the JSON:

```text
Sample Event
     +
Payload Configuration
     ↓
Rendered JSON
```

No external HTTP request fires. Operators validate configuration changes before live delivery feels them.

## Important configuration timing

Configuration types refresh differently:

```text
Routing rules
      ↓
Periodic router refresh

Credentials
      ↓
Credential reload / periodic refresh

Vendor configuration
      ↓
Worker startup / restart

Template fields
      ↓
Worker startup / restart
```

A database edit does not reach every running process instantly. Each configuration domain documents its own refresh semantics.

## Why the worker is generic

The delivery worker follows one standard interface:

```text
Load vendor configuration
        ↓
Receive enriched event
        ↓
Build URL
        ↓
Build JSON
        ↓
Apply authentication
        ↓
Apply rate limiting
        ↓
POST to destination
```

Any vendor expressible in the configuration model reuses this flow, keeping vendor logic out of upstream classification and enrichment.

## Plugins only when configuration is not enough

No configuration engine covers every external API. A plugin abstraction escapes for destinations needing more than:

```text
URL template
+
authentication configuration
+
field mapping
+
transforms
```

```text
Standard API
     ↓
Configuration only

Non-standard API behavior
     ↓
Configuration
     +
Vendor-specific plugin
```

Special cases stay out of the core worker. Plugin registries and names stay out of this article.

## Batching

External APIs batch differently, and vendor configuration says whether events go singly or as an array:

```text
Individual strategy
Event A → HTTP request
Event B → HTTP request
Event C → HTTP request
```

```text
Array strategy
Events A+B+C → one HTTP request
```

One worker implementation serves both expectations.

## Rate limiting

External APIs cap throughput, so the worker rate-limits before calling out:

```text
Kafka events
     │
     ▼
Payload builder
     │
     ▼
Rate limiter
     │
     ▼
External API
```

Fixed or adaptive policy per destination configuration. Production limits stay out of this write-up.

## Circuit breakers

An external outage should not unleash an uncontrolled request stream:

```text
Healthy
   ↓
Requests flow

Repeated failures
   ↓
Circuit opens
   ↓
Requests are temporarily rejected
   ↓
Dependency recovery
   ↓
Circuit closes
```

The breaker guards the worker and the destination together. Naming conventions and thresholds stay internal.

## HTTP authentication

Destinations authenticate differently across query parameters, headers, bearer tokens, and other configured mechanisms:

```text
Query parameter
Header
Bearer token
Other configured mechanism
```

Authentication stays delivery configuration, never hard-coded into classification. Credentials, headers, endpoint shapes, and destination auth specifics stay out of this article.

## Why the worker does not own routing

The worker never asks which destination an event deserves:

> Which destination should receive this event?

Routing already answered that. The worker answers the remaining question:

> How do I deliver this event according to this destination's configuration?

Router decides where. Worker decides how.

## What changes when a new destination is added

For a standard JSON API the work reads:

```text
1. Add vendor configuration
2. Add credentials
3. Add payload-template fields
4. Add routing rules
5. Create the vendor Kafka topic
6. Deploy another instance of the generic worker
```

Classification stays put. Enrichment needs no vendor branch for another standard destination. Deployment and resource specifics stay out of this write-up.

## What still requires custom code

Some integrations resist configuration:

```text
non-standard authentication
non-standard request signing
complex multi-request protocols
non-JSON payloads
stateful request construction
vendor-specific transformations that cannot be represented declaratively
```

The architecture layers its answer:

```text
Configuration first
       ↓
Plugin only when necessary
```

The standard path stays simple while odd integrations still fit.

## What stays unchanged when a vendor is added

The boundary holds from classification down:

```text
Classification
      │
      ▼
Internal Event Type
      │
      ▼
Publisher
      │
      ▼
Canonical Kafka Event
      │
      ▼
Router
      │
      ▼
Vendor-specific destination
```

Classification SQL keeps emitting the same internal semantics. The publisher keeps publishing the same canonical event. The router picks destinations. Workers deliver per destination. New standard vendors never turn the pipeline into vendor branches.

## Isolation between destinations

Each destination runs its own worker process and Kafka topic:

Destination A failing and retrying never stops the consumers behind the others. No single delivery queue and worker should sit in front of all vendors.

## Operational trade-offs

### 1. Configuration flexibility versus complexity

Behavior in configuration means fewer deploys, and the configuration model joins the application contract with validation, authorization, auditability, and careful refresh semantics.

### 2. Generic worker versus custom plugins

Generic workers onboard standard integrations cheaply. Unusual APIs still need custom code.

### 3. Independent workers versus more deployments

Per-destination workers isolate failures and scale independently while multiplying running services.

### 4. Runtime configuration versus consistency

Some configuration refreshes live while other loads at startup. Operators need that documented per domain to know when a change takes effect.

## A useful mental model

Three layers carry the event:

The business layer defines what the event means. Routing decides where it goes. Delivery decides how it looks and travels.

## Closing thought

Integration platforms scale when the core pipeline never learns every external API. The durable boundary runs stable internal event, configuration-driven routing, generic delivery worker, destination configuration, external API:

```text
Stable internal event
        ↓
Configuration-driven routing
        ↓
Generic delivery worker
        ↓
Destination-specific configuration
        ↓
External API
```

Configuration covers the common case, plugins cover exceptions, and independent workers contain destination failures. Above all, each new standard integration stays an exercise in configuration and deployment instead of a rewrite of the classification core. The integration count can grow while the business engine stays vendor-blind.

*That closes the onboarding arc. Till then enjoy your life and happy engineering!*
