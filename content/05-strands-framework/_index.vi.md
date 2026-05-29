---
title: "Strands Agents Framework"
date: 2026-05-29
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Strands Agents Framework

## Tổng Quan

**Strands Agents** là một open-source agent framework từ AWS, cung cấp các khối xây dựng để tạo, kết hợp, và triển khai AI agent. Cả bốn agent trong nền tảng MarTech (Marketer, Databricks, CleverTap, TalonOne) đều được xây dựng bằng Strands.

## Tính Năng Chính

### 1. @tool Annotation

Định nghĩa tool một cách khai báo với decorator `@tool`. Framework tự động tạo tool schema và xử lý truyền tham số:

```python
from strands_agents import tool

@tool
def current_time() -> str:
    """Trả về thời gian hiện tại theo định dạng ISO."""
    from datetime import datetime
    return datetime.now().isoformat()
```

### 2. Tích Hợp MCP Native

Strands hỗ trợ MCP client native — tools từ MCP Gateway tích hợp như local tools:

```python
from bedrock_agentcore.mcp import get_gateway_mcp_client

mcp_client = get_gateway_mcp_client("talonone-mcp-tools")

agent = Agent(
    name="TalonOne Agent",
    tools=[current_time, mcp_client],
    ...
)
```

### 3. System Prompt Động Qua SSM

{{% notice info %}}
**Điểm quan trọng**: Hardcode prompt yêu cầu deploy lại mỗi lần muốn điều chỉnh hành vi. Giải pháp là lưu system prompt trong **AWS SSM Parameter Store**.
{{% /notice %}}

```python
system_prompt = config.get("systemPrompt") or default_system_prompt

agent = Agent(
    name="TalonOne Agent",
    description="Một TalonOne promotions agent...",
    system_prompt=system_prompt,  # Tải động từ SSM
    tools=[current_time, mcp_client],
    model=config.get("modelId"),
)
```

Pattern này cho phép thay đổi prompt engineering mà không cần deploy code — cập nhật SSM parameter, và agent sẽ áp dụng trong lần gọi tiếp theo.

### 4. Tất Cả Model Provider

Strands hỗ trợ nhiều model provider qua một interface thống nhất:

- **Amazon Bedrock** (Claude, Llama, Titan, v.v.)
- **Anthropic** direct API
- **OpenAI** (GPT-4, GPT-4o, v.v.)
- **Bất kỳ endpoint tương thích OpenAI nào**

Model ID có thể được cấu hình khi chạy:

```python
agent = Agent(
    model=config.get("modelId"),  # VD: "us.anthropic.claude-sonnet-4-20250514-v1:0"
    ...
)
```

### 5. Hooks Tích Hợp

Strands cung cấp hệ thống hook để can thiệp vào các sự kiện trong lifecycle của agent:

| Hook | Kích Hoạt |
|------|---------|
| `on_tool_start` | Trước khi tool thực thi |
| `on_tool_end` | Sau khi tool hoàn thành |
| `on_agent_start` | Khi agent bắt đầu xử lý |
| `on_agent_end` | Khi agent kết thúc |
| `on_model_response` | Sau mỗi phản hồi model |

Hook có thể dùng để logging, monitoring, theo dõi chi phí, và logic nghiệp vụ tùy chỉnh mà không cần sửa logic cốt lõi của agent.

## Pattern Tạo Agent

Mỗi agent trong nền tảng MarTech đều tuân theo cùng một pattern tạo:

1. **Tools**: MCP Gateway client + built-in utility tools
2. **System Prompt**: Tải động từ SSM Parameter Store (với default fallback được code)
3. **Model ID**: Có thể cấu hình khi chạy để dễ dàng đổi model

Sự nhất quán này giúp codebase dễ dự đoán và dễ bảo trì trên cả bốn agent.
