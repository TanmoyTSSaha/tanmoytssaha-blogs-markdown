---
title: Running a Data Platform on Kubernetes
slug: running-data-platform-kubernetes
date: 2026-09-27
author: Tanmoy Saha
tags:
  - series
  - data-engineering
  - kubernetes
  - devops
  - architecture
  - real-time
description: How the pipeline's workloads split across Deployments and CronJobs, and why control and data paths stay apart.
reading_time: 18
draft: false
series: Near Real-Time Marketing Pipeline
series_slug: near-real-time-marketing-pipeline
series_order: 12
---
A production data platform usually runs two distinct paths:
```text
Control Path
    ↓
Configuration, credentials, audit, operational controls
Data Path
    ↓
Ingestion → Classification → Enrichment → Delivery
```
The administrative API manages configuration and audit records. Processing and delivery workloads read that configuration, while no classified event ever passes through the API. Operational configuration and high-volume event processing stay decoupled.

<a href="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq?diagram=ZijaOPad2KEhKMOVkEyh">View on Eraser<br /><img src="https://app.eraser.io/workspace/2IoevTvZwqYpEQhiLpGq/preview?diagram=ZijaOPad2KEhKMOVkEyh&type=embed" /></a>

## High-level architecture

```text
                     Admin UI
                        │
                        ▼
                    Admin API
                        │
                        ▼
                Relational Database
                 config + audit
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
   Pipeline CronJob   Enrichment        Retry Scheduler
          │             │                   │
          ▼             ▼                   ▼
     Analytical      Identifier        Canonical Kafka
      Processing      Lookup               Flow
          │             │                   │
          └───────┬─────┴───────────────────┘
                  ▼
           Vendor-specific topics
                  │
                  ▼
           Delivery Workers
                  │
                  ▼
            External APIs
                  │
                  ▼
                 DLQ
```

The platform runs as independent workloads, not one monolithic application.

## Why Kubernetes fits the platform

The system mixes execution models. Some components are long-running consumers:

```text
Enrichment Service
Delivery Worker
Admin API
```

Others are scheduled jobs:

```text
Classification Pipeline
DLQ Retry
Data Export
Partition Maintenance
```

Kubernetes hands each kind the right primitive:

```text
Deployment
    ↓
Long-running service

CronJob
    ↓
Scheduled processing
```

A classification pipeline should not stay alive waiting for its next window, while a Kafka consumer should never sleep.

## Workload decomposition

| Workload | Kubernetes type | Responsibility |
|---|---|---|
| Admin API | Deployment | Configuration, audit, administrative operations |
| Classification Pipeline | CronJob | Windowed classification and publishing |
| Enrichment Service | Deployment | External identifier enrichment and routing |
| Delivery Workers | Deployments | Destination-specific API delivery |
| DLQ Retry Scheduler | CronJob | Automatic recovery of eligible failures |
| Data Export | CronJob | Export operational data to object storage |
| Partition Maintenance | CronJob | Maintain analytical data retention |
| Admin UI | Separate application | Configuration and operational console |

Exact workload names and counts shift as the platform evolves, so those stay out of this write-up.

## Reusing one worker image

One generic delivery-worker image deploys many times over:

```text
Generic Worker Image
        │
        ├── Deployment A
        │      └── Destination A configuration
        │
        ├── Deployment B
        │      └── Destination B configuration
        │
        └── Deployment C
               └── Destination C configuration
```

The binary never changes. Only runtime configuration does. Vendor behavior stays configuration-driven, and every destination keeps its own failure and scaling boundary. A new standard destination reuses the same image instead of growing a new codebase.

## Control path versus data path

Administration manages configuration:

```text
Admin UI
    ↓
Admin API
    ↓
Relational Configuration Store
```

The data path reads the result independently:

```text
Kafka
  ↓
Classification
  ↓
Enrichment
  ↓
Routing
  ↓
Delivery
```

No event calls the administrative API mid-flight. A configuration API serves low-volume admin traffic while the data path moves millions of events on its own.

## Configuration propagation

Workloads notice configuration changes at different moments:

```text
Configuration Store
       │
       ├── Classification
       │      ↓
       │   next processing window
       │
       ├── Enrichment / Routing
       │      ↓
       │   periodic refresh
       │
       └── Delivery Workers
              ↓
           startup / reload
```

Updates are not globally instant, which suits deterministic processing better than flipping every component at the same instant. A classification window keeps the snapshot it started with while routing refreshes separately.

## Administrative API

The admin API fronts the control plane:

- classification configuration;
- routing rules;
- destination configuration;
- credentials;
- payload templates;
- audit logs;
- operational controls.

It talks to the relational configuration database and the analytical store where needed. Internal hostnames, URL paths, namespaces, ingress domains, and network specifics stay out of this article.

## Health and observability endpoints

Long-running services expose health apart from metrics:

```text
Application
   │
   ├── Health endpoint
   │      ├── liveness
   │      └── readiness
   │
   └── Metrics endpoint
          └── Prometheus
```

Kubernetes reads health to decide traffic while Prometheus scrapes measurements on its own. Port numbers, service names, and internal URLs stay out of this description.

## Configuration and audit

Configuration changes stay auditable:

```text
Admin Request
     │
     ▼
Validation
     │
     ▼
Configuration Update
     │
     ├────────► Current Configuration
     │
     └────────► Audit Record
```

An audit record names the table or configuration area, the record identity, the action, the actor, the old and new values, and the timestamp. Operations stay traceable with no source-code archaeology.

## Secrets management

Secrets never live in container images or deployment files:

```text
Secret Store
      │
      ▼
External Secret / Kubernetes Secret
      │
      ▼
Application Pod
```

Plain configuration ships in deployment files while credentials inject at runtime. Secret paths, parameter names, key names, and environment identifiers stay excluded. The platform runs a cloud-native secret flow with external-secret synchronization instead of raw credentials in source control.

## Credentials versus ordinary configuration

```text
Ordinary Configuration
    ↓
URLs
timeouts
batching
routing
feature flags

Secrets
    ↓
API credentials
tokens
encryption keys
```

The deployment system treats the two differently, which keeps credentials out of Git history, container images, logs, manifests, and configuration backups.

## Orchestrator deployment model

The classification pipeline runs as a scheduled Kubernetes CronJob:

```text
CronJob wake
     │
     ▼
Acquire execution ownership
     │
     ▼
Check eligible window
     │
     ├── none ──► exit
     │
     ▼
Process classification window
     │
     ▼
Publish
     │
     ▼
Exit
```

Wake frequency and window size stay separate ideas. A CronJob can wake, find no closed window, and leave. Window semantics stay configurable without teaching the scheduler about them.

## Why concurrency must be controlled

Classification is stateful. Two windows must never update user state together, so Kubernetes scheduling forms one guard while the application adds a database execution lock plus execution-state checks:

```text
Scheduled run
      │
      ▼
Kubernetes concurrency control
      │
      ▼
Application lock
      │
      ▼
Running-state validation
      │
      ▼
Process one window
```

Defense in depth: even a scheduler misfire meets the application lock before touching shared state.

## Enrichment service deployment

Enrichment runs long because it consumes Kafka without pause:

```text
Consume canonical event
        ↓
Resolve external identifiers
        ↓
Apply routing rules
        ↓
Publish destination-specific events
```

Only this service needs the external identifier lookup dependency, which leaves classification sealed off from identifier-store outages.

## Delivery worker deployments

Each external destination gets its own worker deployment:

```text
Destination A Topic → Worker A → API A
Destination B Topic → Worker B → API B
Destination C Topic → Worker C → API C
```

Offsets, retries, scaling, rate limits, circuit breakers, and failure handling all stay independent. One destination's trouble never stops the rest.

## Destination-specific scaling

Long-running workers scale horizontally on Kubernetes:

```text
Kafka lag / CPU / workload
          │
          ▼
        HPA
          │
     ┌────┴────┐
     ▼         ▼
 Worker 1    Worker 2
```

The scaling signal follows the workload and deployment. Replica bounds and thresholds stay environment-specific and out of this article.

## DLQ retry scheduler

The retry scheduler runs as another CronJob. It never calls external APIs itself:

```text
DLQ
 │
 ▼
Retry Scheduler
 │
 ▼
Canonical Kafka Flow
 │
 ▼
Enrichment
 │
 ▼
Routing
 │
 ▼
Destination Worker
```

Automatic recovery walks the same path as normal delivery. No second vendor implementation exists.

## Data export and maintenance jobs

Scheduled maintenance sits beside the platform:

```text
Analytical Data
     │
     ▼
Object Storage Export
```

```text
Analytical Partitions
     │
     ▼
Retention / Cleanup
```

These jobs stay outside the online vendor-delivery path. Maintenance never joins live event routing or API delivery.

## The data stores

### Relational database

Configuration, audit logs, pipeline execution state, DLQ work queue, and operational metadata. Control-plane and operational state, not high-volume analytical scans.

### Analytical database

Transaction ingestion, deduplication, classification, user state, and event-delta generation. The primary analytical engine.

### Kafka

Event ingestion, canonical event transport, destination routing, delivery decoupling, and DLQ and operational event streams. The asynchronous seams between major stages.

### Cassandra / low-latency lookup store

External identifier lookup for enrichment. Classification never needs it.

### Object storage

Exported data, historical processing and reconciliation, and operational archives. Bucket names, account identifiers, paths, and access policies stay omitted.

## Keeping infrastructure details out of application code

Deployments combine three inputs:

```text
Application Image
        +
Environment Configuration
        +
Runtime Secrets
```

One image moves across environments picking up different endpoints, credentials, feature flags, resource settings, Kafka configuration, and database connections. That portability is basic container practice.

## Infrastructure ownership boundaries

Application and infrastructure codebases split cleanly:

```text
Application Repository
    ↓
Application code
Docker images
Helm templates
workload configuration

Infrastructure Repository
    ↓
Cluster resources
Argo CD applications
ExternalSecret resources
environment-specific infrastructure
```

Application teams declare what their service needs. Platform and DevOps teams run the shared cluster. Repository names and organizational domains stay omitted.

## Secrets and external configuration at deployment time

Deployment templates reference external secret sources instead of holding values:

```text
Helm / Kubernetes Configuration
          │
          ▼
External Secret Reference
          │
          ▼
Cloud Secret Store
          │
          ▼
Kubernetes Secret
          │
          ▼
Pod
```

Secret rotation never rebuilds the image. Plain configuration injects the same way through environment variables or files.

## Observability with Prometheus

Long-running workloads expose Prometheus-compatible metrics:

```text
Application
     │
     ▼
 /metrics
     │
     ▼
Prometheus
     │
     ├── dashboards
     └── alerts
```

Useful concepts include processing duration, Kafka throughput, consumer lag, delivery success and failure, DLQ volume, retry counts, and pipeline failures. Metric names, service labels, dashboard URLs, and alert definitions stay unpublished.

## Operational failure domains

The platform partitions into failure domains on purpose. Destination A down means Worker A retries while B and C keep consuming. Cassandra down degrades enrichment while classification state stands. Admin API down freezes configuration changes while running classification continues on its existing snapshot. A local failure should never default into a platform outage.

## What shares a failure domain

Some workloads unavoidably share dependencies:

```text
Classification
   ├── Analytical Database
   ├── Relational Database
   └── Kafka

Enrichment
   ├── Kafka
   ├── Lookup Store
   └── Relational Configuration

Delivery Worker
   ├── Kafka
   ├── Relational Configuration
   └── External API
```

Incident response starts from these groups. One dependency dying never hits every component the same way.

## Deployment lifecycle

```text
Code Change
    │
    ▼
Build Container Image
    │
    ▼
Push Image Registry
    │
    ▼
Update Deployment Configuration
    │
    ▼
GitOps / Deployment Controller
    │
    ▼
Kubernetes
    │
    ▼
New Pods
    │
    ▼
Health Checks
    │
    ▼
Traffic / Processing
```

Registry names, image names, repository paths, and controller configuration stay omitted.

## Why GitOps fits this architecture

GitOps keeps a declarative desired state:

```text
Git
 │
 │ desired state
 ▼
Deployment Controller
 │
 ▼
Kubernetes
 │
 ▼
Running workloads
```

Deployments reproduce, and infrastructure changes review beside source control. Internal infrastructure repositories and application names stay out of this article.

## Operational lessons

### 1. Separate control plane and data plane

Configuration traffic never flows through the high-volume event pipeline.

### 2. Match Kubernetes primitives to workload behavior

Deployments for long-running consumers and APIs. CronJobs for windowed and scheduled processing.

### 3. Reuse generic worker images

One standard delivery worker deploys independently per destination.

### 4. Protect stateful workflows with multiple concurrency controls

Scheduler-level and application-level locking back each other up.

### 5. Keep secrets outside images and source control

Inject at deployment and runtime instead of compiling them in.

### 6. Design failure domains explicitly

Independent workers and Kafka topics stop one external dependency from becoming a global failure point.

### 7. Keep maintenance jobs outside the online path

Exports and partition cleanup never race latency-sensitive event processing.

## A useful mental model

Five layers carry the platform:

```text
┌──────────────────────────────┐
│       Control Plane          │
│ UI / API / Config / Audit    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Data Processing        │
│ Ingestion / Classification   │
│ Dedup / Stateful Processing  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          Routing             │
│ Enrichment / Destination     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          Delivery            │
│ Worker / Rate Limit / Retry  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       External Systems       │
│ APIs / Object Storage / etc. │
└──────────────────────────────┘
```

Underneath all five:

```text
Kubernetes
    +
Kafka
    +
Relational Configuration
    +
Analytical Storage
    +
Observability
```

## Closing thought

Running a data platform on Kubernetes comes down to placement decisions more than containerization:

```text
What should run continuously?
        ↓
What should run periodically?
        ↓
What should scale independently?
        ↓
What configuration does each component need?
        ↓
Which failures should remain isolated?
        ↓
Which state must survive a restart?
```

With those boundaries drawn, Kubernetes executes a set of workloads anyone can understand instead of hosting a monolith that happens to run in containers. Deployment architecture mirrors the execution and failure model of the data platform: long-running consumers, scheduled stateful processing, admin APIs, retry schedulers, and maintenance jobs each get the workload shape and dependency boundaries they need, and the whole platform gets easier to operate, scale, and bring back up.

*That closes the deployment arc. Till then enjoy your life and happy engineering!*
