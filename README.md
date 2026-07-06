![Agentic AI for MarTech - System Architecture](static/MarTech/docs/architecture.jpg)

# AWS Multi-Agent System for MarTech

A bilingual (English / Vietnamese) workshop guide and full-stack reference implementation for building a production-grade **multi-agent AI platform** on AWS - purpose-built for cross-platform marketing campaign orchestration.

---

## The Problem

Marketing teams rely on a fragmented ecosystem of specialized platforms to launch a single campaign. Each platform has its own UI, API contract, authentication model, and domain-specific terminology:

| Platform             | Role                                                                           |
| -------------------- | ------------------------------------------------------------------------------ |
| **Databricks** | Audience analytics - SQL queries, Unity Catalog discovery, job orchestration   |
| **CleverTap**  | Campaign delivery - push, email, SMS, WhatsApp, web push, webhook              |
| **TalonOne**   | Promotions & loyalty - campaigns, coupons, loyalty programs, customer sessions |

A marketer must context-switch between all three, manually transfer data between them, and maintain no end-to-end audit trail. A campaign launch that should take minutes stretches into hours.

---

## The Solution: AI-Guided Multi-Agent Platform

Instead of forcing the marketer to navigate each platform manually, four specialized AI agents collaborate under the direction of a human operator:

### Agent Topology

| Agent                      | Role                 | Key Capabilities                                                                                     | Runtime                                       |
| -------------------------- | -------------------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **Marketing Agent**  | Orchestrator         | Enforces a strict 3-step workflow, delegates via A2A, collects user confirmations, streams SSE to UI | Custom FastAPI + Strands on AgentCore Runtime |
| **Databricks Agent** | Data analytics       | 8 MCP tools - SQL execution, Unity Catalog discovery, job management                                 | A2A Server (Strands) on AgentCore Runtime     |
| **CleverTap Agent**  | Campaign management  | 6 MCP tools - draft-first campaign creation with estimated reach confirmation                        | A2A Server (Strands) on AgentCore Runtime     |
| **TalonOne Agent**   | Promotions & loyalty | 11 MCP tools - campaigns, coupons, loyalty programs, customer sessions                               | A2A Server (Strands) on AgentCore Runtime     |

### Three-Step Campaign Workflow

The Marketing Agent enforces a strict workflow and refuses to help with anything outside it:

1. **Define Target Audience** - The Databricks Agent explores Unity Catalog schemas, runs SQL queries against warehouses, and presents audience segments. This is the most conversation-intensive step (a real session logged 26 messages before audience confirmation).
2. **Create Campaign in CleverTap** - The CleverTap Agent follows a draft-first workflow: it creates a draft with `estimate_only=true`, presents the estimated reach to the user, and only calls `confirm_draft_campaign` after explicit approval.
3. **Create Promotion in TalonOne** *(optional)* - The TalonOne Agent handles campaign creation (defaults to `disabled` for review), coupon management, loyalty point redemption, and customer session updates.

---

## Architecture

### Communication Patterns

Two protocols connect the four agents:

**Agent-to-Agent (A2A) Protocol**

- SigV4-authenticated SSE streaming between agents
- Session ID propagation through the entire call chain
- 4 SSE event types: `text`, `tool_use`, `tool_result`, `subagent_progress`
- IAM-based access control - only the Marketing Agent has permission to invoke worker agents

**Model Context Protocol (MCP) Gateway**

- Centralized route for tool calls from agents to Lambda-based MCP servers
- IAM authentication via SigV4 - no API keys to manage
- Tool name prefix filtering (`databricks-target___execute_sql` → `execute_sql`)
- Agent logic is decoupled from infrastructure - update a Lambda without touching agent code

### Data Flow

```
Web UI → API Gateway (SigV4) → Lambda → Marketing Agent (AgentCore Runtime)
  → A2A → Databricks/CleverTap/TalonOne Agent (AgentCore Runtime)
    → MCP Gateway → Lambda MCP Server (Node.js 22)
      → Third-party API (Databricks/CleverTap/TalonOne)
  → AgentCore Memory (session context)
  → S3 Artifact Hook (audit trail)
```

All four agents write conversation artifacts to the same S3 session folder under their own subdirectory (`orchestrator/`, `databricks-agent/`, `clevertap-agent/`, `talonone-agent/`), creating a complete audit trail across the entire workflow.

---

## AWS Services Powering the Platform

### AgentCore Runtime

Serverless container runtime hosting all 4 agents as Docker images. Each agent runs with a scoped IAM execution role that grants only the permissions it needs (Bedrock model invocation, SSM read, S3 write, A2A invoke).

### AgentCore Memory

Persistent conversation context across multi-turn sessions. The Marketing Agent loads prior decisions on each invocation - enabling long-running campaigns where the user returns after hours or days.

### AgentCore MCP Gateway

Routes tool calls from agents to 3 Lambda-based MCP servers with IAM authentication. Each target exposes between 6 and 11 tools (25 total), with credentials stored in AWS Secrets Manager.

---

## Frontend

The **React 19** single-page application is built with the **Cloudscape Design System** (AWS's open-source design language):

| Technology                 | Purpose                                  |
| -------------------------- | ---------------------------------------- |
| TanStack Router            | Type-safe file-based routing             |
| TanStack React Query       | Server state management                  |
| Cloudscape Chat Components | Real-time chat with SSE streaming        |
| Cognito + OIDC             | Authentication via`react-oidc-context` |
| aws4fetch                  | SigV4 request signing                    |

Routes: Campaign list (`/campaign`), campaign detail with embedded chat (`/campaign/:id`), and configuration page (`/configuration`) for selecting Bedrock models and editing system prompts for all agents - changes take effect immediately via SSM Parameter Store.

---

## Infrastructure (AWS CDK)

The entire platform is defined as TypeScript CDK and deployed as a single CloudFormation stack with 7 constructs:

| Construct                  | Resources                                 | Purpose                                                                           |
| -------------------------- | ----------------------------------------- | --------------------------------------------------------------------------------- |
| **UserIdentity**     | Cognito User Pool + Identity Pool         | OIDC authentication with SigV4 credential exchange                                |
| **StorageAndData**   | DynamoDB table (Campaigns) + 3 S3 buckets | Campaign storage, session artifacts, SQL results, access logs                     |
| **GatewayConstruct** | AgentCore MCP Gateway + 3 Lambda targets  | Routes 25 tools with IAM auth                                                     |
| **AgentConstruct**   | 4 AgentCore Runtimes + shared Memory      | Hosts all agents with scoped IAM roles                                            |
| **APIConstruct**     | API Gateway + 9 Lambda handlers           | REST endpoints for campaigns, chat (15-min streaming), configuration, SQL results |
| **WebUi**            | S3 + CloudFront                           | Static site hosting for the React frontend                                        |
| **SeedConfig**       | SSM Parameter Store                       | Default model IDs and system prompts for all 4 agents                             |

---

## MCP Servers (Lambda, Node.js 22)

Three platform-specific MCP servers, each fetching credentials from Secrets Manager:

| Server                   | Tools | Description                                                                                                                   |
| ------------------------ | ----- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Databricks MCP** | 8     | SQL execution, Unity Catalog discovery, job orchestration. Large results (>20 rows/10KB) auto-uploaded to S3                  |
| **CleverTap MCP**  | 6     | Draft-first campaign creation with audience estimation, multi-channel support (push, email, SMS, WhatsApp, web push, webhook) |
| **TalonOne MCP**   | 11    | Campaigns (CRUD), coupons (list/validate/create), loyalty (balance/redeem), customer sessions                                 |

---

## Strands Agents Framework

All four agents are built with the **open-source Strands Agents Framework** from AWS:

- **`@tool` decorator** - Declarative tool definitions with automatic schema generation
- **Native MCP integration** - `get_gateway_mcp_client()` creates a SigV4-authenticated MCP client filtered by target prefix
- **Dynamic system prompts** - Loaded from SSM Parameter Store at invocation time; prompt changes require no code redeployment
- **Multi-model support** - Amazon Bedrock, Anthropic direct, OpenAI, or any OpenAI-compatible endpoint; model ID configurable at runtime
- **Built-in hooks** - `S3ArtifactHook` intercepts `MessageAddedEvent` to persist conversations to S3

---

## Monorepo Structure

```
packages/
  web-ui/            # React 19 frontend (Vite, Cloudscape, TanStack)
  api/               # 9 Lambda handlers (Node.js, rolldown-bundled)
  infra/             # AWS CDK stack (TypeScript, 7 constructs)
  common/
    constructs/      # Shared CDK constructs
    types/           # Deployment config types
  agents/
    common/          # Shared Python utilities (A2A server, MCP client, config loader, S3 hook)
    marketer/        # Marketing Agent (Python, FastAPI, orchestrator)
    databricks/      # Databricks Agent (Python, Strands A2A server)
    clevertap/       # CleverTap Agent (Python, Strands A2A server)
    talonone/        # TalonOne Agent (Python, Strands A2A server)
```

Build system: **pnpm** workspaces + **Nx** task orchestration for Node.js, **uv** for Python, **Vite** for the frontend, **Rolldown** for Lambda bundling.

---

## Deployment

```bash
pnpm install && uv sync          # Install dependencies
pnpm run build:all               # Build all packages
pnpm exec nx deploy @project/infra "stack-name/*"  # Deploy to AWS
```
