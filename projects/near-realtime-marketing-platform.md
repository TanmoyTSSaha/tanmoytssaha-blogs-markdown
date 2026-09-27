---
id: nrt-marketing-platform
title: "Near Real-Time Performance Marketing Platform"
slug: "nrt-marketing-platform"
date: "2025-06-01"
status: "completed"
type: "company"
tags:
  - Kafka
  - StarRocks
  - Go
  - Kubernetes
  - Spark
description: "Configuration-driven near real-time marketing data platform processing 90M+ user transaction records daily with 30-minute micro-batch processing."
repo_url: ""
demo_url: ""
---

## Overview

Engineered a configuration-driven platform that replaced D-1 marketing data delivery with 30-minute micro-batch processing for 90M+ daily user transaction records at Paytm.

---

## Architecture

- Kafka-to-StarRocks pipeline using Routine Load, with a Kubernetes CronJob-driven Go orchestrator for deduplication and rule-based classification.
- DLQ and retry infrastructure with exponential backoff through vendor-specific Kafka topics: 3-4M records/hour at 1-2 minute end-to-end latency and >95% delivery success.
- S3 export service plus D-1 Spark/Azkaban reconciliation workflow validating classified-versus-delivered counts and republishing retryable failures.
- Go orchestrator, enrichment router, API service, and ReactJS frontend deployed across 10 EKS services.
- MySQL-backed configurations and AWS Secrets Manager credentials enabling dynamic vendor onboarding without pipeline redesigns.

---

## Impact

- Replaced next-day vendor delivery with deterministic 30-minute micro-batches.
- Sustained 3-4M records/hour with >95% delivery success.
