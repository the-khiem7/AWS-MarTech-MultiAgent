---
title: "Strands Agents Framework"
date: 2026-05-29
weight: 7
chapter: false
pre: " <b> 7. </b> "
---

# Strands Agents Framework

**Strands Agents** là một open-source agent framework từ AWS, cung cấp các khối xây dựng để tạo, kết hợp, và triển khai AI agent. Cả bốn agent trong nền tảng MarTech đều được xây dựng bằng Strands.

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

mcp_client = get_gateway_mcp_client("talonone-target")

agent = Agent(
    name="TalonOne Agent",
    tools=[current_time, mcp_client],
    ...
)
```

Factory `get_gateway_mcp_client` tạo MCP client xác thực SigV4, kết nối đến gateway qua streamable HTTP và lọc tools theo prefix `{target_name}___`.

### 3. System Prompt Động Qua SSM

{{% notice info %}}
**Điểm quan trọng**: Hardcode prompt yêu cầu deploy lại mỗi lần muốn điều chỉnh hành vi. Nền tảng MarTech lưu system prompt trong **AWS SSM Parameter Store**, và tất cả agent tải chúng tại thời điểm gọi.
{{% /notice %}}

```python
config = load_configuration()  # Đọc SSM parameter từ biến môi trường AGENT_CONFIG_PARAMETER

agent = Agent(
    name="TalonOne Agent",
    system_prompt=config.get("systemPrompt") or default_system_prompt,
    tools=[current_time, mcp_client],
    model=config.get("modelId"),
)
```

Hàm `load_configuration()` trong `packages/agents/common/common/config.py` lấy JSON config từ SSM. Nếu parameter không được đặt, không tìm thấy, hoặc không tải được, nó trả về dict rỗng — fallback về default được code. Pattern này cho phép thay đổi prompt engineering mà không cần deploy code.

Web UI cung cấp trang **Configuration** cho phép người dùng chọn Bedrock model và tùy chỉnh system prompt cho từng agent qua `GET /configuration/{agentName}` và `PUT /configuration/{agentName}` — cả hai đều đọc/ghi SSM Parameter Store.

### 4. Tất Cả Model Provider

Strands hỗ trợ nhiều model provider qua một interface thống nhất:

- **Amazon Bedrock** (Claude, Llama, Titan, v.v.)
- **Anthropic** direct API
- **OpenAI** (GPT-4, GPT-4o, v.v.)
- **Bất kỳ endpoint tương thích OpenAI nào**

Model ID có thể cấu hình khi chạy từ SSM:

```python
agent = Agent(
    model=config.get("modelId"),  # VD: "us.anthropic.claude-sonnet-4-20250514-v1:0"
    ...
)
```

Dropdown chọn model trong Web UI được populate bởi `GET /configuration/models`, liệt kê các Bedrock foundation models và inference profiles khả dụng.

### 5. Hooks Tích Hợp

Strands cung cấp hệ thống hook để can thiệp vào các sự kiện trong lifecycle của agent:

| Hook | Kích Hoạt |
|------|---------|
| `on_tool_start` | Trước khi tool thực thi |
| `on_tool_end` | Sau khi tool hoàn thành |
| `on_agent_start` | Khi agent bắt đầu xử lý |
| `on_agent_end` | Khi agent kết thúc |
| `on_model_response` | Sau mỗi phản hồi model |

Nền tảng MarTech đăng ký `S3ArtifactHook` trên `MessageAddedEvent` của mỗi agent để lưu tất cả tin nhắn hội thoại vào S3. Hook cũng có thể dùng để logging, monitoring, theo dõi chi phí, và logic nghiệp vụ tùy chỉnh — tất cả mà không cần sửa logic cốt lõi của agent.

## Pattern Tạo Agent

Mỗi agent trong nền tảng MarTech đều tuân theo cùng một pattern tạo:

| Thành Phần | Pattern |
|-----------|---------|
| **Tools** | MCP Gateway client (lọc theo target prefix) + built-in `current_time` |
| **System Prompt** | Tải động từ SSM Parameter Store qua `load_configuration()` với default fallback được code |
| **Model ID** | Có thể cấu hình khi chạy từ SSM — trang Configuration của Web UI cho phép đổi model mà không cần redeploy |

Sự nhất quán này giúp codebase dễ dự đoán và dễ bảo trì trên cả bốn agent. Shared utilities trong `packages/agents/common/` loại bỏ trùng lặp code cho các mối quan tâm chung: A2A server setup, gateway MCP clients, configuration loading, và S3 artifact hooks.
