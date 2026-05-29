# Workshop guildline

## Giới thiệu

* [https://advertising.amazon.com/vi-vn/library/guides/what-is-martech#1](https://advertising.amazon.com/vi-vn/library/guides/what-is-martech#1)

## PainPoint

* Mkt-er muốn launch một campaint.
  * Phải querry audience qua Databrick
  * ClerverTap quản lí campaign
  * [Talon.One](http://Talon.One) quản lí promotion

## Solution: AI Guided workflow

* Mỗi platform được một agent đảm nhiệm
  * Databrick Agent
  * ClerverTap Agent
  * TalonOne Agent

![architecture](static/MarTech/docs/architecture.jpg)

## AWS Agentic Service

### AgentCore Runtime

* Give an Docker Image, AWS handle the rest

  > HTTPS Connect use SigV4 auth instead sendding API Key in https
  >
* 

### AgentCore Memory

* Làm bộ nhớ cho agent quá mỗi lượt gọi
* Phân tích session id, prompt id, . . .
* Có thể truy xuất ngược vào conversation mới

### AgentCore MCP Gateway

* Khi mỗi MCP được deploy voà Lambda
* MCP Gateway sẽ giúp quán lí các MCP trong kiến trúc AWS
* Xác thực bằng SigV4 IAM
* Có thể thay đổi Lambda mà không động vào logic của agent

**Workflow**: Agent -> MCP Gateway -> MCP Lambda -> Agent -> APIs

**Code**: [mcp_client = get_gateway_mcp_client(mcpname-mcptools)](../static/MarTech/packages/agents/talonone/app/agent/agent.py)

```python
    system_prompt = config.get("systemPrompt") or default_system_prompt

    return Agent(
        name="TalonOne Agent",
        description="A TalonOne promotions agent for campaigns, loyalty programs, coupons, and customer sessions.",
        system_prompt=system_prompt,
        tools=[current_time, mcp_client],
        model=config.get("modelId"),
        callback_handler=None,
    )
```

### A2A

* Một cổng giao tiếp xem agents khác là một tool (như mcp)
* Sử dụng Framework Stands A2AAgent với xác thực SigV4
* Session ID được gán qua chuỗi tool call
* Thông tin tiến trình được truyền như SSE event

**Code**: [async def databricks_agent(request: str) -&gt; AsyncIterator](../static/MarTech/packages/agents/marketer/app/agent/worker_agents/databricks.py)

```python
def build_databricks_tool(agent_runtime_arn: str, region: str, session_id: str):
    """Create a tool that delegates Databricks tasks to the remote agent."""

    @tool
    async def databricks_agent(request: str) -> AsyncIterator:
        """Send a data analytics request to the Databricks agent.

        Use this tool for any Databricks-related tasks including:
        - Executing SQL queries against Databricks warehouses
        - Discovering schemas, tables, and columns in Unity Catalog
        - Running and monitoring Databricks jobs
        - Audience segmentation and data analysis

        Args:
            request: A natural language description of the data task.
        """
        async for event in stream_a2a_agent(
            agent_runtime_arn,
            region,
            request,
            session_id,
        ):
            yield event

    return databricks_agent
```

### Strand Agents Framework

**Open source agent framework from AWS**

* @tool anotation
* native mcp using
* builtin hook
* all model provider

### Agent Pattern

* Tools: MCP Gateway + builtin
* System Prompt: From SSM + Parameter Store
* Model ID: can Config at runtime

#### PainPoint

> Các prompt khi hardcoded vào code, khi cần sửa prompt thì phải deploy lại.
>
> Giải pháp là đưa system prompt vào SSM Param Store để có thể áp dụng được config động

## Cloudscape UI Kit

Một UI Framework có phong cách tương tự AWS Console, thường dùng cho các dự án thiên về cloud hoặc muốn follow theo aws
