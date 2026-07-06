---
title: "Platform Agents & MCP Servers"
date: 2026-05-29
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

Three platform-specific agents handle Databricks, CleverTap, and TalonOne operations. Each connects to a dedicated Lambda-based MCP server behind the AgentCore MCP Gateway for tool execution.

---

## Databricks Agent

**Location**: `packages/agents/databricks/`

**Architecture**: Python FastAPI (via shared `create_a2a_app` factory) → Strands Agent → MCP Gateway Client (prefix: `databricks-target___`) → Databricks MCP Lambda

**Agent Configuration**:

```python
def get_databricks_agent():
    mcp_client = get_gateway_mcp_client("databricks-target")
    config = load_configuration()

    return Agent(
        name="Databricks Agent",
        description="Data analytics agent",
        system_prompt=config.get("systemPrompt"),
        tools=[current_time, mcp_client],
        model=config.get("modelId"),
    )
```

### MCP Server Tools (8)

| Tool | Databricks API | Description |
|------|---------------|-------------|
| `execute_sql` | POST `/api/2.0/sql/statements` | Execute SQL with optional catalog, schema, row_limit |
| `get_statement_result` | GET `/api/2.0/sql/statements/{id}` | Poll for results of pending/running statements |
| `list_warehouses` | GET `/api/2.0/sql/warehouses` | List all available SQL warehouses |
| `list_schemas` | GET `/api/2.1/unity-catalog/schemas` | List schemas in a catalog |
| `list_tables` | GET `/api/2.1/unity-catalog/tables` | List tables in a schema |
| `get_table` | GET `/api/2.1/unity-catalog/tables/{name}` | Get table details (columns, types) |
| `run_job` | POST `/api/2.1/jobs/run-now` | Trigger a Databricks job run |
| `get_job_run` | GET `/api/2.1/jobs/runs/get` | Check job run status |

**Credentials**: Stored in Secrets Manager as `{ url, token }` (workspace URL + Personal Access Token).

**SQL Result Truncation**: Results exceeding 20 rows or 10KB are uploaded to S3. The response includes `_truncated.total_rows`, `_truncated.preview_rows`, and `_truncated.full_result_s3_uri`. The Web UI provides a download link via `GET /sql-result/{key}`.

### Workflow Guidelines

The agent's system prompt instructs it to:
1. Use `list_warehouses` to find available warehouses
2. Use `list_schemas` and `list_tables` to discover data before writing queries
3. Use `get_table` to understand column names and types before constructing SQL
4. Poll with `get_statement_result` if queries are still pending
5. Inform the user about S3 locations when results are truncated

---

## CleverTap Agent

**Location**: `packages/agents/clevertap/`

**Architecture**: Python FastAPI (via shared `create_a2a_app` factory) → Strands Agent → MCP Gateway Client (prefix: `clevertap-target___`) → CleverTap MCP Lambda

### MCP Server Tools (6)

| Tool | CleverTap API | Description |
|------|--------------|-------------|
| `create_draft_campaign` | POST `/1/targets/create.json` | Validate with `estimate_only=true`; returns estimated reach |
| `confirm_draft_campaign` | POST `/1/targets/create.json` | Create campaign with `estimate_only=false` |
| `list_draft_campaigns` | POST `/1/targets/list.json` | List campaigns in date range (YYYYMMDD) |
| `get_draft_campaign` | POST `/1/targets/result.json` | Get full details of a draft |
| `update_draft_campaign` | POST `/1/targets/create.json` | Re-validate draft with updated parameters |
| `discard_draft_campaign` | POST `/1/targets/stop.json` | Permanently delete a draft |

**Credentials**: Stored in Secrets Manager as `{ projectId, passcode, region }`.

**Audience Targeting**: The server translates `user_property_filters` (array of `{ name, operator, value }`) into CleverTap's `common_profile_properties.profile_fields` query language. Supports optional event-based filters with `event_name`, `from`, and `to` date range.

**Channels**: push, email, SMS, webpush, whatsapp, webhook.

### Draft-First Workflow

1. Gather required info: name, channel, content, and `user_property_filters`
2. Always `create_draft_campaign` first - never send without a draft
3. Present estimated reach and ask for confirmation
4. If confirmed → `confirm_draft_campaign`; if changes needed → `update_draft_campaign`
5. If cancelled → `discard_draft_campaign`

---

## TalonOne Agent

**Location**: `packages/agents/talonone/`

**Architecture**: Python FastAPI (via shared `create_a2a_app` factory) → Strands Agent → MCP Gateway Client (prefix: `talonone-target___`) → TalonOne MCP Lambda

### MCP Server Tools (11)

| Domain | Tool | API | Description |
|--------|------|-----|-------------|
| **Campaigns** | `list_campaigns` | Management API | List with state filter and pagination |
| | `get_campaign` | Management API | Get campaign details by ID |
| | `create_campaign` | Management API | Create new campaign (defaults `disabled`) |
| **Sessions** | `get_customer_session` | Management API | Get sessions by profile ID |
| | `update_customer_session` | Integration API | Update/create session with cart items |
| **Loyalty** | `get_loyalty_program` | Management API | Get program details or list all |
| | `get_customer_loyalty` | Management API | Get loyalty ledger balances |
| | `redeem_points` | Management API | Deduct points from balance |
| **Coupons** | `list_coupons` | Management API | List coupons for a campaign |
| | `validate_coupon` | Management API | Search by coupon code across campaigns |
| | `create_coupon` | Management API | Create coupon (code, discount type, value) |

**Credentials**: Stored in Secrets Manager as `{ baseUrl, applicationId, managementKey, integrationKey }`. The Management API uses `ManagementKey-v1` auth; the Integration API uses `ApiKey-v1` auth.

### Workflow Guidelines

1. Use `list_campaigns` before other operations
2. Use `get_customer_session` before making updates
3. Check loyalty status with `get_customer_loyalty` before redeeming points
4. Validate coupons with `validate_coupon` before applying
5. Find a campaign_id via `list_campaigns` before creating coupons
6. New campaigns default to `disabled` for review

---

## Shared Agent Utilities

All agents share common Python utilities in `packages/agents/common/common/`:

| Module | Purpose |
|--------|---------|
| `a2a_server.py` | `create_a2a_app()` factory - wraps a Strands Agent in an A2A Server with FastAPI |
| `gateway.py` | `get_gateway_mcp_client(target_name)` - creates SigV4-authenticated MCP client with tool prefix filtering |
| `config.py` | `load_configuration()` - reads agent config (modelId, systemPrompt) from SSM Parameter Store |
| `s3_artifact.py` | `S3ArtifactHook` - writes conversation messages to S3 for audit |

{{% notice tip %}}
The Marketing Agent uses a custom FastAPI server with a `/invocations` endpoint (required by AgentCore Runtime) instead of the shared `create_a2a_app` factory, because it acts as the orchestrator rather than a worker.
{{% /notice %}}
