---
title: "A2A & Session Persistence"
date: 2026-05-29
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

The Agent-to-Agent (A2A) protocol and session persistence mechanism are deeply intertwined. Understanding how session IDs propagate through A2A calls is key to understanding how all four agents maintain a coherent conversation.

---

## A2A Communication

**What it is**: A protocol that allows one agent to call another as if it were a local tool. The Marketing Agent uses A2A to delegate to Databricks, CleverTap, and TalonOne agents.

### A2A Streaming

The Marketing Agent defines each worker agent as a `@tool`-decorated async generator using `stream_a2a_agent()`:

```python
from strands_agents.a2a import stream_a2a_agent

def build_databricks_tool(agent_runtime_arn: str, region: str, session_id: str):
    @tool
    async def databricks_agent(request: str) -> AsyncIterator:
        async for event in stream_a2a_agent(
            agent_runtime_arn,
            region,
            request,
            session_id,
        ):
            yield event

    return databricks_agent
```

The `stream_a2a_agent` function:
1. Fetches the remote agent's card via `boto3 GetAgentCard`
2. Constructs a Strands `A2AAgent` with a **SigV4-authenticated HTTPX client**
3. Streams responses as `SubAgentProgress` events for intermediate updates + final result string

### IAM Access Control

The Marketing Agent's IAM execution role explicitly grants:
- `bedrock-agentcore:InvokeAgentRuntime` on each worker's ARN
- `bedrock-agentcore:GetAgentCard` on each worker's ARN

No other agent can call another - only the orchestrator has A2A permissions.

{{% notice tip %}}
`stream_a2a_agent` handles the entire A2A handshake - SigV4 signing, connection establishment, and SSE event parsing - so you only need the target agent's ARN and region.
{{% /notice %}}

---

## Session Persistence

All four agents write their conversation artifacts to the same S3 session folder. This creates a complete audit trail of every agent interaction across the workflow.

### S3 Artifact Structure

From a real session (`session-115d83b0-d13a-436b-af69-63e556b601a9`):

```
/<sessions-bucket>/
└── session-115d83b0-d13a-436b-af69-63e556b601a9/
    ├── orchestrator/
    │   ├── agent.json
    │   └── messages/
    │       ├── message_0.json
    │       ├── message_1.json
    │       ...
    ├── databricks-agent/
    │   ├── agent.json
    │   └── messages/
    │       ├── message_0.json
    │       ...
    └── clevertap-agent/
        ├── agent.json
        └── messages/
            ├── message_0.json
            ...
```

In this session, the user went through Step 1 (26 Databricks messages) and Step 2 (4 CleverTap messages). The TalonOne agent was not invoked.

### How Session ID Propagates

The session ID flows through the entire call chain:

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant Lambda as Put Chat Lambda
    participant Mkt as Marketing Agent
    participant DB as Databricks Agent
    participant S3 as S3 Bucket
    UI->>Lambda: PUT /chat {sessionId: "abc"}
    Lambda->>Mkt: x-amzn-bedrock-agentcore-runtime-session-id: abc
    Note over Mkt: Sets current_session_id = abc
    Mkt->>DB: A2A call with X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: abc
    Note over DB: SessionIdMiddleware reads header, sets current_session_id
    Mkt->>S3: Write orchestrator/message_*.json
    DB->>S3: Write databricks-agent/message_*.json
{{< /mermaid >}}

**Key mechanisms**:

1. The Put Chat Lambda passes the session ID via the `x-amzn-bedrock-agentcore-runtime-session-id` header to the Marketing Agent's AgentCore Runtime
2. The Marketing Agent's `main.py` reads this header and sets a `current_session_id` context variable
3. Each A2A tool builder captures the `session_id` in its closure and passes it to `stream_a2a_agent()`, which sets it as the `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` header on the HTTP request to each worker agent
4. Worker agents have a `SessionIdMiddleware` in their FastAPI app that reads the header and sets `current_session_id`
5. The `S3ArtifactHook` reads `current_session_id` on each `MessageAddedEvent` and writes to the correct subfolder

### S3ArtifactHook

The shared `S3ArtifactHook` in `packages/agents/common/common/s3_artifact.py` is used by all four agents:

| Agent | agent_id | S3 Path |
|-------|----------|---------|
| Marketing Agent | `orchestrator` | `<session_id>/orchestrator/messages/` |
| Databricks Agent | `databricks-agent` | `<session_id>/databricks-agent/messages/` |
| CleverTap Agent | `clevertap-agent` | `<session_id>/clevertap-agent/messages/` |
| TalonOne Agent | `talonone-agent` | `<session_id>/talonone-agent/messages/` |

The hook handles:
- Lazy session initialization (creates `agent.json` on first message)
- Message indexing (counts existing messages to determine the next index)
- Content serialization (converts Strands `ContentBlock` objects to JSON)
- Graceful failure (logs warnings but never blocks agent execution)

{{% notice info %}}
**S3 is write-only for session persistence.** Session restoration is handled by AgentCore Memory, not by reading from S3. The S3 artifacts exist for audit, debugging, and inspection.
{{% /notice %}}

### SSE Event Format

The Marketing Agent's streaming handler emits four SSE event types that the Web UI renders in real-time:

| Event Type | Payload | When Emitted |
|------------|---------|--------------|
| `text` | `{ content: "..." }` | Text chunks from the agent |
| `tool_use` | `{ name: "...", input: {...} }` | When a tool invocation starts |
| `tool_result` | `{ name: "...", status: "...", output: "..." }` | When a tool completes |
| `subagent_progress` | `{ agent: "...", content: "..." }` | Intermediate streaming from worker agents |

The Chat component in the Web UI renders each type differently - text with Markdown support, tool use/results as expandable panels, and subagent progress attached to the parent tool use block.
