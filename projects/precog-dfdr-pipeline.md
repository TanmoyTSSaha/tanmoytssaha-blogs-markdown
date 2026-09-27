---
id: precog-dfdr-pipeline
title: "PRECOG -- Flight DFDR Data Pipeline"
slug: "precog-dfdr-pipeline"
date: "2024-06-01"
status: "completed"
type: "company"
tags:
  - PySpark
  - AWS
  - EMR
  - S3
description: "PySpark-based flight DFDR processing workflows on AWS processing ~3,000 files daily with 66 exceedance parameters for operational reporting."
repo_url: ""
demo_url: ""
---

## Overview

Built and optimized PySpark-based flight DFDR processing workflows on AWS EMR, S3, Glue, Athena, EventBridge, and Step Functions at SpiceJet, processing approximately 3,000 files daily.

---

## Architecture

- Multi-stage PySpark workflows on EMR ingesting ~3,000 DFDR files daily.
- 66 exceedance parameters for operational analysis and monitoring.
- Data quality and governance checks guarding downstream analytics.
- Analytical datasets and exceedance reports served to operations teams via Athena and S3.

---

## Impact

- Reliable daily DFDR processing at scale with clean, governed datasets.
- Operational reporting adopted by operations teams for day-to-day decisions.
