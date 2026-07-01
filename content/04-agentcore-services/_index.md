---
title: "AWS AgentCore Services"
date: 2026-05-29
weight: 4
chapter: false
pre: " <b> 4. </b> "
---

AWS AgentCore is the managed service backbone powering the entire multi-agent system. It provides three core capabilities that eliminate the undifferentiated heavy lifting of building production AI agent infrastructure.

---

## AgentCore Runtime

**What it is**: A serverless container runtime that hosts AI agents. You provide a Docker image, and AWS handles deployment, scaling, health checks, and observability.

**How the Platform Uses It**:

The MarTech platform deploys **four AgentCore Runtimes** via the `AgentConstruct` CDK construct:

| Agent Runtime | Entry Pattern | IAM Role Permissions |
|---------------|---------------|---------------------|
| **Marketing Agent** | Custom FastAPI with `/invocations` endpoint | Bedrock model invocation, SSM read, Memory full access, S3 read/write, A2A invoke + GetAgentCard for all workers |
| **Databricks Agent** | Shared A2A server factory (`create_a2a_app`) | Bedrock model invocation, SSM read, Gateway invocation |
| **CleverTap Agent** | Shared A2A server factory | Bedrock model invocation, SSM read, Gateway invocation |
| **TalonOne Agent** | Shared A2A server factory | Bedrock model invocation, SSM read, Gateway invocation |

Each agent construct creates an IAM execution role assumed by `bedrock-agentcore.amazonaws.com` with inline policies scoped to the specific agent's needs.

{{% notice tip %}}
Using SigV4 authentication instead of API keys means your agents never need to manage or rotate credentials manually — IAM handles it automatically.
{{% /notice %}}

---

## AgentCore Memory

**What it is**: A persistent memory layer that retains conversation context across multiple turns and sessions.

**How the Platform Uses It**:

The `AgentConstruct` creates a shared **AgentCore Memory** resource (`marketer_memory`) used by the Marketing Agent. The Memory ID is passed via the `MEMORY_ID` environment variable.

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as Put Chat Lambda
    participant Mkt as Marketing Agent
    participant Mem as AgentCore Memory
    participant DB as Databricks Agent
    UI->>API: "Launch campaign for premium users"
    API->>Mkt: Invoke with session_id=abc
    Mkt->>Mem: Load context for session abc
    Mkt->>DB: A2A: query audience
    DB-->>Mkt: 50K users found
    Mkt->>Mem: Save audience context
    Mkt-->>UI: "Found 50K users. Confirm?"
    UI->>API: "Yes, create campaign"
    API->>Mkt: Invoke with session_id=abc
    Mkt->>Mem: Recall: "session abc, step 1 confirmed"
    Mkt->>Mkt: Continue to step 2...
{{< /mermaid >}}

The Marketing Agent configures Memory via `AgentCoreMemoryConfig` and `AgentCoreMemorySessionManager`. The Chat History API handler (`GET /chat/:sessionId`) reads from Memory using `ListEventsCommand` to reconstruct conversation history for the UI.

{{% notice info %}}
AgentCore Memory is distinct from the S3 artifact storage. Memory handles session restoration for the agent; S3 artifacts are for audit and inspection. The S3 hook is write-only.
{{% /notice %}}

---

## AgentCore MCP Gateway

**What it is**: A managed gateway that routes tool calls from agents to Lambda-based MCP (Model Context Protocol) servers.

**How the Platform Uses It**:

The `GatewayConstruct` creates the gateway and registers **three Lambda targets**:

| Target | Tools | Lambda Config | Credentials Source |
|--------|-------|---------------|-------------------|
| **DatabricksTarget** | 8 | Node.js 22.x, 60s timeout, 256MB | Secrets Manager (url, token) |
| **ClevertapTarget** | 6 | Node.js 22.x, 30s timeout, 256MB | Secrets Manager (projectId, passcode, region) |
| **TalonOneTarget** | 11 | Node.js 22.x, 30s timeout, 256MB | Secrets Manager (baseUrl, applicationId, managementKey, integrationKey) |

**Tool name convention**: The gateway prefixes tool names with the target name and three underscores (`databricks-target___execute_sql`). The shared `extractToolName()` utility strips this prefix so agents see clean tool names. The gateway MCP client filters tools by the `{target_name}___` regex pattern to ensure each agent only sees its own tools.

```python
from bedrock_agentcore.mcp import get_gateway_mcp_client

mcp_client = get_gateway_mcp_client("talonone-target")
# Agent only sees tools prefixed with "talonone-target___"
```

### MCP Server Details

Each MCP server is a Lambda function that:

1. Receives a tool call from the gateway with a `GatewayContext` (tool name, message version, request ID, gateway ID, target ID)
2. Fetches platform credentials from **AWS Secrets Manager** (cached for the Lambda execution context lifetime)
3. Translates the tool call into the platform's native REST API
4. Returns structured results

{{% notice warning %}}
The Databricks MCP server has special handling for large results: SQL results exceeding 20 rows or 10KB are truncated. The preview is returned inline, but the full dataset is uploaded to the **SQL Results S3 bucket** as JSON. The response includes a `_truncated` object with `total_rows`, `preview_rows`, `full_result_s3_uri`, and `full_result_bytes`.
{{% /notice %}}

---

## How They Work Together

| Layer | Service | How the Platform Uses It |
|-------|---------|------------------------|
| Hosting | AgentCore Runtime | 4 Docker-based agent containers with IAM roles |
| Memory | AgentCore Memory | Shared memory for Marketing Agent session context |
| Tools | AgentCore MCP Gateway | 3 Lambda targets exposing 25 tools with IAM auth |

This separation allows each layer to evolve independently — update a Lambda tool without touching agent code, swap the model without changing infrastructure, or add a new platform agent without modifying existing ones.
