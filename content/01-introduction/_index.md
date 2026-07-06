---
title: "Introduction"
date: 2026-05-29
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

## The Modern Marketing Challenge

To launch a single marketing campaign, a marketer must coordinate across three disconnected platforms:

- **Databricks** - query audience segments, run SQL against data warehouses, discover schemas and tables in Unity Catalog
- **CleverTap** - manage campaign lifecycle: create drafts, validate targeting, confirm delivery across push, email, SMS, WhatsApp, web push, and webhook channels
- **TalonOne** - configure promotions, coupons, loyalty programs, customer sessions, and manage discount campaigns

Each platform requires separate credentials, different API contracts, and domain-specific expertise. The marketer spends more time context-switching between tools than designing effective campaigns.

{{% notice info %}}
**Real cost**: A campaign launch that should take minutes stretches into hours, with error-prone manual data transfer between platforms.
{{% /notice %}}

## Solution: AI-Guided Multi-Agent Platform

Instead of forcing the marketer to navigate each platform manually, we introduce an **AI-powered multi-agent system** where four specialized agents collaborate to execute campaigns end-to-end:

- A **Marketing Agent** orchestrate a strict 3-step campaign creation workflow, guiding the user at each decision point
- A **Databricks Agent** explores data catalogs, runs SQL queries, and segments audiences
- A **CleverTap Agent** creates draft campaigns, estimates reach, and confirms delivery after user approval
- A **TalonOne Agent** manages promotions, loyalty programs, coupons, and customer sessions (optional step)

The marketer simply chats with the orchestrator agent in natural language. The platform handles all cross-system coordination behind the scenes.

## What You'll Build

By the end of this workshop, you'll understand:

| Component | What You'll Learn |
|-----------|------------------|
| **Multi-Agent Workflow** | How 4 agents coordinate through a 3-step campaign creation process with user confirmation at each stage |
| **AWS AgentCore** | How AgentCore Runtime, Memory, and MCP Gateway eliminate infrastructure complexity |
| **Platform Agents & MCP** | How 3 Lambda-based MCP servers expose 25+ tools across Databricks, CleverTap, and TalonOne |
| **A2A Communication** | How agents delegate tasks to each other with SigV4 authentication, SSE streaming, and session persistence |
| **Strands Framework** | How the open-source Strands Agents framework provides @tool decorators, hooks, MCP integration, and dynamic prompts |
| **Web UI & Infrastructure** | How a React/Cloudscape frontend connects to 9 Lambda handlers, 4 AgentCore endpoints, and CDK-provisioned infrastructure |

## AWS AgentCore: The Foundation

The entire system runs on **AWS AgentCore** - a managed service purpose-built for deploying and operating AI agents at scale. It provides three core capabilities:

| Service | Role in This Platform |
|---------|----------------------|
| **AgentCore Runtime** | Hosts all 4 agents as Docker containers with automatic scaling, SigV4 authentication, and built-in observability |
| **AgentCore Memory** | Persists conversation context across messages, enabling the Marketing Agent to recall prior decisions in long-running campaigns |
| **AgentCore MCP Gateway** | Routes tool calls from agents to 3 Lambda-based MCP servers with IAM authentication - decoupling agent logic from infrastructure |
