---
title: "Architecture Overview"
date: 2026-05-29
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Architecture Overview

## System Architecture

The MarTech multi-agent platform follows a layered architecture where specialized AI agents interact with marketing platforms through a unified orchestration layer.

![System Architecture](_static/architecture.jpg)

## Agent Topology

Four agents collaborate to execute marketing campaigns:

| Agent | Responsibility | Platform | Framework |
|-------|---------------|----------|-----------|
| **Marketer Agent** | Orchestrator — receives user intent, delegates to sub-agents | AWS AgentCore Runtime | Strands Agents |
| **Databricks Agent** | Audience segmentation, SQL queries, data analysis | Databricks Unity Catalog + Warehouses | Strands Agents + MCP |
| **CleverTap Agent** | Campaign creation, notification delivery, user tracking | CleverTap | Strands Agents + MCP |
| **TalonOne Agent** | Promotions, coupons, loyalty programs, customer sessions | TalonOne | Strands Agents + MCP |

## Data Flow

When a marketer describes a campaign in natural language:

1. **Marketer Agent** receives the request and determines which sub-agents are needed
2. **Databricks Agent** queries audience segments and returns target customer profiles
3. **CleverTap Agent** creates the campaign with the identified audience
4. **TalonOne Agent** attaches applicable promotions and discounts
5. Results bubble back to the **Marketer Agent**, which reports to the user

## Communication Patterns

Two communication patterns are used:

{{< tabs >}}

{{< tab name="MCP Gateway" >}}
Each platform agent (Databricks, CleverTap, TalonOne) exposes its capabilities as **MCP tools** via the **AgentCore MCP Gateway**. Agents invoke MCP tools directly, and the gateway handles:
- IAM SigV4 authentication
- Routing to the correct Lambda-based MCP server
- Decoupling agent logic from infrastructure
{{< /tab >}}

{{< tab name="A2A (Agent-to-Agent)" >}}
The Marketer Agent communicates with platform agents via **A2A**, treating each sub-agent as a remote tool. A2A provides:
- SigV4-authenticated streaming via SSE events
- Session ID propagation across tool call chains
- Progress reporting back to the orchestrator
{{< /tab >}}

{{< /tabs >}}

## Frontend

A React-based web UI built with **Cloudscape Design System** — AWS's design language — provides:
- Campaign management dashboard
- Real-time chat interface with the Marketer Agent
- Cognito authentication and user management

The frontend communicates with a TypeScript Lambda backend via REST APIs (API Gateway), which proxies requests to the agent runtime.
