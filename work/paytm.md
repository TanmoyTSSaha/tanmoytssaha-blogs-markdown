---
id: paytm-data-engineer
company: "Paytm"
role: "Data Engineer -- Central Data Platform"
location: "Noida, Uttar Pradesh, India"
type: "Full-time"
start_date: "2025-02-01"
end_date: "Present"
is_current: true
company_url: "https://paytm.com/"
logo_url: "https://cdn.iconscout.com/icon/free/png-512/free-paytm-icon-svg-download-png-226448.png"
technologies:
  - Python
  - Go
  - SQL
  - Spark
  - Kafka
  - StarRocks
  - AWS
  - MySQL
  - Cassandra
  - Kubernetes
  - Docker
  - Azkaban
  - ArgoCD
  - Prometheus
  - Grafana
  - ReactJS
description: "Data Engineer on the Central Data Platform, building near real-time marketing data infrastructure processing 90M+ user transaction records daily, plus AI-powered engineering automation."
---

## Overview

At Paytm, I work on the Central Data Platform — building distributed data platforms, near real-time pipelines, and AI-powered engineering automation. My work spans cost and performance optimization of batch infrastructure as well as greenfield near real-time systems serving performance marketing at 90M+ user transaction records daily.

---

## Cost & Performance Optimization

- Reduced S3 storage by 45% by auditing and optimizing lifecycle policies, removing obsolete data and non-functional policies based on storage usage patterns.
- Improved Spark job performance by narrowing data scans and eliminating redundant transformations, reducing execution time and compute requirements.

---

## Near Real-Time Performance Marketing Platform

Built and deployed a configuration-driven near real-time marketing data platform processing 90M+ user transaction records daily, replacing D-1 vendor delivery with deterministic 30-minute micro-batch processing.

- Designed the Kafka-to-StarRocks pipeline using Routine Load and a Kubernetes CronJob-driven Go orchestrator for deduplication and rule-based classification before publishing classified records to Kafka.
- Built the DLQ and retry infrastructure, classifying failures by retryability and implementing exponential-backoff retries through vendor-specific Kafka topics; processed 3-4M records/hour with 1-2 minute end-to-end latency and >95% delivery success.
- Developed the S3 export service and D-1 Spark/Azkaban reconciliation workflow to validate classified-versus-delivered counts, identify retryable failures, and republish eligible records for recovery.
- Contributed to the Go orchestrator, enrichment router, API service, and ReactJS frontend, and deployed the complete application stack across 10 EKS services.
- Implemented configuration-driven vendor payload generation using MySQL-backed configurations and AWS Secrets Manager for application credentials, enabling dynamic vendor onboarding without redesigning the core pipeline.

---

## MCP-Powered Real-Time Event Onboarding Automation

Built a Python MCP server using Streamable HTTP to automate onboarding of real-time BAU events, reducing onboarding time from 2-3 hours to approximately 30 minutes.

- Automated Jira validation, Confluent Schema Registry registration, YAML configuration generation using Jira and Confluence, Bitbucket changes, and Kubernetes deployment orchestration through Argo CD.
- Integrated the workflow with Cursor, Claude, and internal AI agents, with Prometheus-based deployment tracking and Slack notifications for progress and failures; automated 30-40 deployments with 100% deployment success.

---

## Impact

- Replaced D-1 marketing data delivery with 30-minute micro-batch processing over 90M+ daily records.
- Cut real-time event onboarding from 2-3 hours to ~30 minutes with fully automated deployments.
- Saved infrastructure cost through S3 lifecycle optimization and Spark performance tuning.

---

## Technologies Deep Dive

### Data Engineering
- Python
- Go
- SQL
- Spark
- Kafka
- StarRocks
- MySQL
- Cassandra
- Azkaban

### Infrastructure
- AWS (S3, EKS, EC2, EMR, Lambda, RDS, Secrets Manager)
- Docker
- Kubernetes
- ArgoCD
- BitBucket

### Observability
- Prometheus
- Grafana
- Alerting via email and Slack

### Fullstack
- ReactJS
- REST APIs
