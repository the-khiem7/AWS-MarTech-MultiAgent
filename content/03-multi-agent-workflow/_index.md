---
title: "Multi-Agent Workflow"
date: 2026-05-29
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

# Multi-Agent Workflow

The Marketing Agent enforces a strict three-step campaign creation workflow. The user must confirm each step before proceeding — the agent refuses to help with anything outside this workflow.

## Step 1 — Define Target Audience

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as API Handler
    participant Mkt as Marketing Agent
    participant DB as Databricks Agent
    participant GW as MCP Gateway
    participant DBR as Databricks
    UI->>API: User sends prompt
    API->>Mkt: Invoke via AgentCore Runtime
    Mkt->>DB: Delegate audience task (A2A)
    DB->>GW: Call Databricks tools
    GW->>DBR: API calls (SQL, catalog discovery)
    DBR-->>GW: Results
    GW-->>DB: Tool results
    DB-->>Mkt: Audience data
    Mkt-->>UI: Stream audience details (SSE)
    Note over Mkt,UI: User reviews and confirms audience
{{< /mermaid >}}

The **Databricks Agent** handles this step using 8 MCP tools:

| Tool | Purpose |
|------|---------|
| `list_warehouses` | Discover available SQL warehouses |
| `list_schemas` | Explore Unity Catalog schemas |
| `list_tables` | List tables within a schema |
| `get_table` | Inspect column names and types |
| `execute_sql` | Run SQL queries against a warehouse |
| `get_statement_result` | Poll for long-running query results |
| `run_job` | Trigger a Databricks job |
| `get_job_run` | Check job run status |

**Workflow**: The agent explores available data sources, constructs and executes SQL queries, and presents audience segments. SQL results exceeding 20 rows or 10KB are uploaded to S3 as JSON — the agent informs the user about the download location.

{{% notice info %}}
This is the most conversation-intensive step. A real session logged 26 messages between the user, Marketing Agent, and Databricks Agent before reaching audience confirmation.
{{% /notice %}}

## Step 2 — Create Campaign in CleverTap

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as API Handler
    participant Mkt as Marketing Agent
    participant CT as CleverTap Agent
    participant GW as MCP Gateway
    participant CTR as CleverTap
    UI->>API: User confirms audience
    API->>Mkt: Invoke via AgentCore Runtime
    Mkt->>CT: Delegate campaign creation (A2A)
    CT->>GW: Call CleverTap tools
    GW->>CTR: API calls (draft, confirm)
    CTR-->>GW: Results
    GW-->>CT: Tool results
    CT-->>Mkt: Campaign details
    Mkt-->>UI: Stream campaign confirmation (SSE)
{{< /mermaid >}}

The **CleverTap Agent** enforces a draft-first workflow:

| Tool | Description |
|------|-------------|
| `create_draft_campaign` | Validate targeting with `estimate_only=true`; returns estimated reach |
| `confirm_draft_campaign` | Actually create the campaign with `estimate_only=false` |
| `list_draft_campaigns` | List campaigns in a date range |
| `get_draft_campaign` | Get details of a specific draft |
| `update_draft_campaign` | Re-validate a draft with updated parameters |
| `discard_draft_campaign` | Permanently delete a draft |

**Channels supported**: push, email, SMS, web push, WhatsApp, webhook. For email, SMS, and WhatsApp, a `provider_nick_name` is required.

{{% notice warning %}}
The agent **always** creates a draft first. It never creates a campaign directly without user confirmation of the estimated reach.
{{% /notice %}}

## Step 3 — Create Promotion in TalonOne (Optional)

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as API Handler
    participant Mkt as Marketing Agent
    participant TO as TalonOne Agent
    participant GW as MCP Gateway
    participant TOR as TalonOne
    UI->>API: User opts into promotion
    API->>Mkt: Invoke via AgentCore Runtime
    Mkt->>TO: Delegate promotion task (A2A)
    TO->>GW: Call TalonOne tools
    GW->>TOR: API calls (campaigns, coupons)
    TOR-->>GW: Results
    GW-->>TO: Tool results
    TO-->>Mkt: Promotion details
    Mkt-->>UI: Stream promotion confirmation (SSE)
{{< /mermaid >}}

The **TalonOne Agent** provides 11 tools across four domains:

| Domain | Tools |
|--------|-------|
| **Campaigns** | `list_campaigns`, `get_campaign`, `create_campaign`|
| **Loyalty** | `get_loyalty_program`, `get_customer_loyalty`, `redeem_points` |
| **Coupons** | `list_coupons`, `validate_coupon`, `create_coupon`|
| **Sessions** | `get_customer_session`, `update_customer_session`|

Campaigns default to `disabled` state for review before activation. The agent discovers available campaigns with `list_campaigns` before performing any other operation.

## Behind the Scenes

While the user sees a simple chat, three critical mechanisms run invisibly:

1. **AgentCore Memory** retains conversation context between messages, so the Marketing Agent remembers which step the user is on and what was previously confirmed.

2. **Session ID Propagation** — the same session ID flows from the Web UI → Put Chat Lambda → Marketing Agent → A2A calls → Worker Agents → S3. All four agents write to the same S3 folder.

3. **SSE Streaming** — the Marketing Agent emits four event types (`text`, `tool_use`, `tool_result`, `subagent_progress`) that the Web UI renders in real-time, giving the user visibility into tool invocations and intermediate agent reasoning.
