---
title: "Dịch Vụ AWS AgentCore"
date: 2026-05-29
weight: 4
chapter: false
pre: " <b> 4. </b> "
---

AWS AgentCore là xương sống dịch vụ quản lý cho toàn bộ hệ thống multi-agent. Nó cung cấp ba khả năng cốt lõi giúp loại bỏ công việc nặng nhọc khi xây dựng hạ tầng AI agent production.

---

## AgentCore Runtime

**Là gì**: Một container runtime serverless để host AI agent. Bạn cung cấp Docker image, AWS xử lý việc triển khai, scaling, health check, và observability.

**Cách Nền Tảng Sử Dụng**:

Nền tảng MarTech triển khai **bốn AgentCore Runtime** qua CDK construct `AgentConstruct`:

| Agent Runtime | Entry Pattern | IAM Role Permissions |
|---------------|---------------|---------------------|
| **Marketing Agent** | Custom FastAPI với endpoint `/invocations` | Bedrock model invocation, SSM read, Memory full access, S3 read/write, A2A invoke + GetAgentCard cho tất cả worker |
| **Databricks Agent** | Shared A2A server factory (`create_a2a_app`) | Bedrock model invocation, SSM read, Gateway invocation |
| **CleverTap Agent** | Shared A2A server factory | Bedrock model invocation, SSM read, Gateway invocation |
| **TalonOne Agent** | Shared A2A server factory | Bedrock model invocation, SSM read, Gateway invocation |

Mỗi agent construct tạo một IAM execution role được assume bởi `bedrock-agentcore.amazonaws.com` với inline policies được giới hạn theo nhu cầu cụ thể của từng agent.

{{% notice tip %}}
Dùng SigV4 thay vì API key có nghĩa là agent của bạn không bao giờ phải quản lý hoặc xoay vòng credentials thủ công - IAM xử lý tự động.
{{% /notice %}}

---

## AgentCore Memory

**Là gì**: Một lớp bộ nhớ liên tục giữ lại ngữ cảnh hội thoại qua nhiều lượt và phiên.

**Cách Nền Tảng Sử Dụng**:

`AgentConstruct` tạo một **AgentCore Memory** dùng chung (`marketer_memory`) cho Marketing Agent. Memory ID được truyền qua biến môi trường `MEMORY_ID`.

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as Put Chat Lambda
    participant Mkt as Marketing Agent
    participant Mem as AgentCore Memory
    participant DB as Databricks Agent
    UI->>API: "Triển khai chiến dịch cho người dùng premium"
    API->>Mkt: Gọi với session_id=abc
    Mkt->>Mem: Tải context cho session abc
    Mkt->>DB: A2A: truy vấn audience
    DB-->>Mkt: Tìm thấy 50K người dùng
    Mkt->>Mem: Lưu context audience
    Mkt-->>UI: "Tìm thấy 50K người dùng. Xác nhận?"
    UI->>API: "Có, tạo chiến dịch"
    API->>Mkt: Gọi với session_id=abc
    Mkt->>Mem: Nhớ lại: "session abc, bước 1 đã xác nhận"
    Mkt->>Mkt: Tiếp tục bước 2...
{{< /mermaid >}}

Marketing Agent cấu hình Memory qua `AgentCoreMemoryConfig` và `AgentCoreMemorySessionManager`. Chat History API handler (`GET /chat/:sessionId`) đọc từ Memory bằng `ListEventsCommand` để tái tạo lịch sử hội thoại cho UI.

{{% notice info %}}
AgentCore Memory khác với S3 artifact storage. Memory xử lý session restoration cho agent; S3 artifacts dành cho audit và kiểm tra. S3 hook là write-only.
{{% /notice %}}

---

## AgentCore MCP Gateway

**Là gì**: Một gateway quản lý định tuyến tool call từ agent đến Lambda-based MCP (Model Context Protocol) server.

**Cách Nền Tảng Sử Dụng**:

`GatewayConstruct` tạo gateway và đăng ký **ba Lambda targets**:

| Target | Tools | Lambda Config | Nguồn Credentials |
|--------|-------|---------------|-------------------|
| **DatabricksTarget** | 8 | Node.js 22.x, 60s timeout, 256MB | Secrets Manager (url, token) |
| **ClevertapTarget** | 6 | Node.js 22.x, 30s timeout, 256MB | Secrets Manager (projectId, passcode, region) |
| **TalonOneTarget** | 11 | Node.js 22.x, 30s timeout, 256MB | Secrets Manager (baseUrl, applicationId, managementKey, integrationKey) |

**Quy ước tên tool**: Gateway thêm prefix tên target và ba dấu gạch dưới (`databricks-target___execute_sql`). Utility `extractToolName()` dùng chung loại bỏ prefix này để agent thấy tên tool sạch. Gateway MCP client lọc tools theo regex pattern `{target_name}___` để đảm bảo mỗi agent chỉ thấy tools của riêng mình.

```python
from bedrock_agentcore.mcp import get_gateway_mcp_client

mcp_client = get_gateway_mcp_client("talonone-target")
# Agent chỉ thấy tools có prefix "talonone-target___"
```

### Chi Tiết MCP Server

Mỗi MCP server là một Lambda function:

1. Nhận tool call từ gateway với `GatewayContext` (tool name, message version, request ID, gateway ID, target ID)
2. Lấy platform credentials từ **AWS Secrets Manager** (cache trong suốt vòng đời Lambda execution context)
3. Dịch tool call thành REST API của nền tảng
4. Trả về kết quả có cấu trúc

{{% notice warning %}}
Databricks MCP server có xử lý đặc biệt cho kết quả lớn: SQL results vượt 20 dòng hoặc 10KB bị cắt ngắn. Preview được trả về inline, nhưng toàn bộ dataset được upload lên **SQL Results S3 bucket** dạng JSON. Response bao gồm object `_truncated` với `total_rows`, `preview_rows`, `full_result_s3_uri`, và `full_result_bytes`.
{{% /notice %}}

---

## Cách Chúng Hoạt Động Cùng Nhau

| Lớp | Dịch Vụ | Cách Nền Tảng Sử Dụng |
|-------|---------|------------------------|
| Hosting | AgentCore Runtime | 4 Docker-based agent containers với IAM roles |
| Bộ Nhớ | AgentCore Memory | Memory dùng chung cho Marketing Agent session context |
| Tools | AgentCore MCP Gateway | 3 Lambda targets cung cấp 25 tools với IAM auth |

Sự phân tách này cho phép mỗi lớp phát triển độc lập - cập nhật Lambda tool mà không cần sửa code agent, đổi model mà không cần thay đổi hạ tầng, hoặc thêm platform agent mới mà không ảnh hưởng đến các agent hiện có.
