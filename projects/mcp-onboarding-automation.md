---
id: mcp-onboarding-automation
title: "MCP-Powered Real-Time Event Onboarding Automation"
slug: "mcp-onboarding-automation"
date: "2025-09-01"
status: "completed"
type: "company"
tags:
  - Python
  - MCP
  - Kubernetes
  - ArgoCD
description: "Python MCP server automating real-time event onboarding across Jira, schema registry, and Kubernetes, cutting onboarding from 2-3 hours to ~30 minutes."
repo_url: ""
demo_url: ""
---

## Overview

Built a Python MCP server using Streamable HTTP to automate real-time event onboarding across Jira validation, schema registration, YAML configuration, Bitbucket changes, and Kubernetes deployment at Paytm.

---

## Architecture

- Jira validation and Confluent Schema Registry registration automated per event.
- YAML configuration generation using Jira and Confluence sources.
- Bitbucket changes and Kubernetes deployment orchestration through Argo CD.
- Integration with Cursor, Claude, and internal AI agents.
- Prometheus-based deployment tracking with Slack notifications for progress and failures.

---

## Impact

- Reduced onboarding time from 2-3 hours to approximately 30 minutes.
- Automated 30-40 deployments with 100% deployment success.
