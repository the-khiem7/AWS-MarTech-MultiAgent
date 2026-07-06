---
title: "Agentic AI for MarTech"
date: 2026-05-29
weight: 1
chapter: false
---

# Agentic AI for MarTech

An AI-powered marketing campaign management platform built on AWS, enabling marketers to orchestrate campaigns across Databricks, CleverTap, and TalonOne through an intelligent multi-agent system.

## Executive Summary

Marketing teams rely on a fragmented ecosystem of specialized platforms. Launching a single campaign requires navigating Databricks for audience analytics, CleverTap for campaign delivery, and TalonOne for promotions and loyalty. Each platform demands separate credentials, different API contracts, and domain-specific expertise. The result: a process that should take minutes stretches into hours, with error-prone manual data transfer between systems.

**Agentic AI for MarTech** solves this by introducing an intelligent multi-agent system that unifies the entire workflow behind a single chat interface. Four specialized AI agents collaborate autonomously — guided by human approval at each decision point — to execute campaigns end-to-end.

## Business Value

| Metric | Before | After |
|--------|--------|-------|
| Campaign launch time | Hours (manual context-switching) | Minutes (guided chat) |
| Cross-platform coordination | Manual copy-paste | Automated A2A orchestration |
| Audit trail | Scattered, non-existent | Unified S3 artifacts per session |
| Agent behavior changes | Code redeployment required | Instant via SSM parameter updates |
| Credential management | Per-platform API keys | Centralized in AWS Secrets Manager |

## Solution Overview

The platform deploys four AI agents on **AWS AgentCore** — a managed service purpose-built for production AI agents:

![System Architecture](02-architecture/_static/architecture.jpg)

The **Marketing Agent** acts as an intelligent orchestrator, enforcing a structured three-step workflow:

1. **Audience Discovery** — The Databricks Agent explores Unity Catalog schemas, runs SQL queries, and presents audience segments for approval.
2. **Campaign Creation** — The CleverTap Agent drafts the campaign with estimated reach, then confirms delivery only after the user authorizes it.
3. **Promotion Setup** *(optional)* — The TalonOne Agent creates promotion campaigns, manages coupons, and handles loyalty point redemption.

At every step, the system waits for human confirmation. Agents never act autonomously without consent.

## Technical Foundation

- **Runtime**: 4 serverless Docker containers (Marketing, Databricks, CleverTap, TalonOne) with scoped IAM roles and SigV4 authentication
- **Communication**: Agent-to-Agent (A2A) protocol with SSE streaming and session ID propagation across the entire call chain
- **Tool Access**: 25 Model Context Protocol (MCP) tools across 3 Lambda-based servers, routed through the AgentCore MCP Gateway
- **Frontend**: React 19 with Cloudscape Design System, TanStack Router, and real-time chat via streaming SSE events
- **Infrastructure**: Single AWS CDK stack (TypeScript) — API Gateway, 9 Lambda handlers, DynamoDB, Cognito, S3, CloudFront
- **Agent Framework**: Open-source Strands Agents framework with `@tool` decorators, dynamic SSM prompts, multi-provider model support, and lifecycle hooks

## Why This Matters

This is not a theoretical exercise. The architecture is designed for real-world production use:

- **Platform-agnostic** — The MCP pattern makes adding new platforms (Salesforce, HubSpot, etc.) a matter of writing a new Lambda server and Gateway target
- **Cost-efficient** — Estimated ~$76-186/month for development usage with pay-per-use AWS services
- **Observable** — Every agent message is persisted to S3 across all four agents, creating a complete, auditable conversation trail
- **Flexible** — System prompts and model selection are configurable at runtime via SSM Parameter Store, with zero code redeployment required

## Next Steps

1. Review the [prerequisites and cost estimates](09-prerequisites) to prepare your environment
2. Follow the [hands-on exercises](10-exercises) to deploy the stack and run a live campaign
3. Explore the [architecture deep-dive](02-architecture) for implementation details
