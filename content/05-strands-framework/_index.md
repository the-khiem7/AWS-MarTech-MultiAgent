---
title: "Strands Agents Framework"
date: 2026-05-29
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Strands Agents Framework

## Overview

**Strands Agents** is an open-source agent framework from AWS that provides the building blocks for creating, composing, and deploying AI agents. All four agents in the MarTech platform (Marketer, Databricks, CleverTap, TalonOne) are built with Strands.

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

mcp_client = get_gateway_mcp_client("talonone-mcp-tools")

agent = Agent(
    name="TalonOne Agent",
    tools=[current_time, mcp_client],
    ...
)
```

### 3. Dynamic System Prompts via SSM

{{% notice info %}}
**Key insight**: Hardcoding prompts requires redeployment every time you want to tune behavior. The solution is storing system prompts in **AWS SSM Parameter Store**.
{{% /notice %}}

```python
system_prompt = config.get("systemPrompt") or default_system_prompt

agent = Agent(
    name="TalonOne Agent",
    description="A TalonOne promotions agent...",
    system_prompt=system_prompt,  # Dynamically loaded from SSM
    tools=[current_time, mcp_client],
    model=config.get("modelId"),
)
```

This pattern allows prompt engineering changes without any code deployment — update the SSM parameter, and the agent picks it up on the next invocation.

### 4. All Model Providers

Strands supports multiple model providers through a unified interface:

- **Amazon Bedrock** (Claude, Llama, Titan, etc.)
- **Anthropic** direct API
- **OpenAI** (GPT-4, GPT-4o, etc.)
- **Any OpenAI-compatible endpoint**

The model ID can be configured at runtime:

```python
agent = Agent(
    model=config.get("modelId"),  # e.g., "us.anthropic.claude-sonnet-4-20250514-v1:0"
    ...
)
```

### 5. Built-in Hooks

Strands provides a hook system for intercepting agent lifecycle events:

| Hook | Trigger |
|------|---------|
| `on_tool_start` | Before a tool executes |
| `on_tool_end` | After a tool completes |
| `on_agent_start` | When an agent begins processing |
| `on_agent_end` | When an agent finishes |
| `on_model_response` | After each model response |

Hooks can be used for logging, monitoring, cost tracking, and custom business logic without modifying the agent's core logic.

## Agent Creation Pattern

Every agent in the MarTech platform follows the same creation pattern:

1. **Tools**: MCP Gateway client + built-in utility tools
2. **System Prompt**: Loaded dynamically from SSM Parameter Store (with a coded default fallback)
3. **Model ID**: Configurable at runtime for easy model switching

This consistency makes the codebase predictable and maintainable across all four agents.
