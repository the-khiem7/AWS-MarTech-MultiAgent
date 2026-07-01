---
title: "Strands Agents Framework"
date: 2026-05-29
weight: 7
chapter: false
pre: " <b> 7. </b> "
---

**Strands Agents** is an open-source agent framework from AWS that provides the building blocks for creating, composing, and deploying AI agents. All four agents in the MarTech platform are built with Strands.

## Key Features

### 1. @tool Annotation

Define tools declaratively with the `@tool` decorator. The framework automatically generates tool schemas and handles argument marshalling:

```python
from strands_agents import tool

@tool
def current_time() -> str:
    """Return the current time in ISO format."""
    from datetime import datetime
    return datetime.now().isoformat()
```

### 2. Native MCP Integration

Strands supports MCP clients natively — tools from the MCP Gateway integrate just like local tools:

```python
from bedrock_agentcore.mcp import get_gateway_mcp_client

mcp_client = get_gateway_mcp_client("talonone-target")

agent = Agent(
    name="TalonOne Agent",
    tools=[current_time, mcp_client],
    ...
)
```

The `get_gateway_mcp_client` factory creates a SigV4-authenticated MCP client that connects to the gateway via streamable HTTP and filters tools by the `{target_name}___` prefix.

### 3. Dynamic System Prompts via SSM

{{% notice info %}}
**Key insight**: Hardcoding prompts requires redeployment every time you want to tune behavior. The MarTech platform stores system prompts in **AWS SSM Parameter Store**, and all agents load them at invocation time.
{{% /notice %}}

```python
config = load_configuration()  # Reads SSM parameter from AGENT_CONFIG_PARAMETER env var

agent = Agent(
    name="TalonOne Agent",
    system_prompt=config.get("systemPrompt") or default_system_prompt,
    tools=[current_time, mcp_client],
    model=config.get("modelId"),
)
```

The `load_configuration()` utility in `packages/agents/common/common/config.py` fetches the JSON config from SSM. If the parameter is not set, not found, or fails to load, it returns an empty dict — falling back to coded defaults. This pattern allows prompt engineering changes without any code deployment.

The Web UI provides a **Configuration page** that lets users select a Bedrock model and customize the system prompt for each agent via `GET /configuration/{agentName}` and `PUT /configuration/{agentName}` — both of which read/write SSM Parameter Store.

### 4. All Model Providers

Strands supports multiple model providers through a unified interface:

- **Amazon Bedrock** (Claude, Llama, Titan, etc.)
- **Anthropic** direct API
- **OpenAI** (GPT-4, GPT-4o, etc.)
- **Any OpenAI-compatible endpoint**

The model ID can be configured at runtime from SSM:

```python
agent = Agent(
    model=config.get("modelId"),  # e.g., "us.anthropic.claude-sonnet-4-20250514-v1:0"
    ...
)
```

The Web UI's model selector dropdown is populated by `GET /configuration/models`, which lists available Bedrock foundation models and inference profiles.

### 5. Built-in Hooks

Strands provides a hook system for intercepting agent lifecycle events:

| Hook | Trigger |
|------|---------|
| `on_tool_start` | Before a tool executes |
| `on_tool_end` | After a tool completes |
| `on_agent_start` | When an agent begins processing |
| `on_agent_end` | When an agent finishes |
| `on_model_response` | After each model response |

The MarTech platform registers an `S3ArtifactHook` on each agent's `MessageAddedEvent` to persist all conversation messages to S3. Hooks can also be used for logging, monitoring, cost tracking, and custom business logic — all without modifying the agent's core logic.

## Agent Creation Pattern

Every agent in the MarTech platform follows the same creation pattern:

| Component | Pattern |
|-----------|---------|
| **Tools** | MCP Gateway client (filtered by target prefix) + built-in `current_time` |
| **System Prompt** | Loaded dynamically from SSM Parameter Store via `load_configuration()` with coded default fallback |
| **Model ID** | Configurable at runtime from SSM — the Web UI Configuration page lets users change models without redeployment |

This consistency makes the codebase predictable and maintainable across all four agents. The shared utilities in `packages/agents/common/` eliminate code duplication for common concerns: A2A server setup, gateway MCP clients, configuration loading, and S3 artifact hooks.
