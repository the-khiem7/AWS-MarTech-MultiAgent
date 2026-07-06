---
title: "A2A & Lưu Trữ Phiên"
date: 2026-05-29
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

Giao thức Agent-to-Agent (A2A) và cơ chế lưu trữ phiên liên kết chặt chẽ với nhau. Hiểu cách session ID lan truyền qua các A2A call là chìa khóa để hiểu cách cả bốn agent duy trì một hội thoại nhất quán.

---

## Giao Tiếp A2A

**Là gì**: Một giao thức cho phép một agent gọi agent khác như thể đó là local tool. Marketing Agent dùng A2A để phân công cho Databricks, CleverTap, và TalonOne agent.

### A2A Streaming

Marketing Agent định nghĩa mỗi worker agent như một `@tool`-decorated async generator sử dụng `stream_a2a_agent()`:

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

Hàm `stream_a2a_agent`:
1. Lấy agent card từ xa qua `boto3 GetAgentCard`
2. Xây dựng Strands `A2AAgent` với **HTTPX client xác thực SigV4**
3. Stream responses dưới dạng `SubAgentProgress` events cho cập nhật trung gian + chuỗi kết quả cuối cùng

### IAM Access Control

IAM execution role của Marketing Agent cấp quyền rõ ràng:
- `bedrock-agentcore:InvokeAgentRuntime` trên ARN của từng worker
- `bedrock-agentcore:GetAgentCard` trên ARN của từng worker

Không agent nào khác có thể gọi agent khác - chỉ orchestrator có quyền A2A.

{{% notice tip %}}
`stream_a2a_agent` xử lý toàn bộ quy trình bắt tay A2A - ký SigV4, thiết lập kết nối, và phân tích SSE events - bạn chỉ cần ARN và region của agent đích.
{{% /notice %}}

---

## Lưu Trữ Phiên

Cả bốn agent ghi artifact hội thoại của mình vào cùng một thư mục S3 session. Điều này tạo ra một audit trail hoàn chỉnh cho mọi tương tác agent trong workflow.

### Cấu Trúc S3 Artifact

Từ một phiên thực tế (`session-115d83b0-d13a-436b-af69-63e556b601a9`):

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

Trong phiên này, người dùng đã qua Bước 1 (26 tin nhắn Databricks) và Bước 2 (4 tin nhắn CleverTap). TalonOne agent không được gọi.

### Cách Session ID Lan Truyền

Session ID chảy qua toàn bộ chuỗi gọi:

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant Lambda as Put Chat Lambda
    participant Mkt as Marketing Agent
    participant DB as Databricks Agent
    participant S3 as S3 Bucket
    UI->>Lambda: PUT /chat {sessionId: "abc"}
    Lambda->>Mkt: x-amzn-bedrock-agentcore-runtime-session-id: abc
    Note over Mkt: Đặt current_session_id = abc
    Mkt->>DB: A2A call với X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: abc
    Note over DB: SessionIdMiddleware đọc header, đặt current_session_id
    Mkt->>S3: Ghi orchestrator/message_*.json
    DB->>S3: Ghi databricks-agent/message_*.json
{{< /mermaid >}}

**Cơ chế chính**:

1. Put Chat Lambda truyền session ID qua header `x-amzn-bedrock-agentcore-runtime-session-id` đến AgentCore Runtime của Marketing Agent
2. `main.py` của Marketing Agent đọc header này và đặt `current_session_id` context variable
3. Mỗi A2A tool builder capture `session_id` trong closure và truyền cho `stream_a2a_agent()`, hàm này đặt nó làm header `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` trên HTTP request đến từng worker agent
4. Worker agent có `SessionIdMiddleware` trong FastAPI app đọc header và đặt `current_session_id`
5. `S3ArtifactHook` đọc `current_session_id` trên mỗi `MessageAddedEvent` và ghi vào subfolder tương ứng

### S3ArtifactHook

`S3ArtifactHook` dùng chung trong `packages/agents/common/common/s3_artifact.py` được cả bốn agent sử dụng:

| Agent | agent_id | Đường Dẫn S3 |
|-------|----------|---------|
| Marketing Agent | `orchestrator` | `<session_id>/orchestrator/messages/` |
| Databricks Agent | `databricks-agent` | `<session_id>/databricks-agent/messages/` |
| CleverTap Agent | `clevertap-agent` | `<session_id>/clevertap-agent/messages/` |
| TalonOne Agent | `talonone-agent` | `<session_id>/talonone-agent/messages/` |

Hook xử lý:
- Khởi tạo phiên lười (tạo `agent.json` ở tin nhắn đầu tiên)
- Đánh chỉ mục tin nhắn (đếm tin nhắn hiện có để xác định chỉ mục tiếp theo)
- Serialize nội dung (chuyển Strands `ContentBlock` thành JSON)
- Thất bại mềm (log cảnh báo nhưng không bao giờ chặn thực thi agent)

{{% notice info %}}
**S3 là write-only cho lưu trữ phiên.** Session restoration được xử lý bởi AgentCore Memory, không phải đọc từ S3. S3 artifacts tồn tại cho audit, debug, và kiểm tra.
{{% /notice %}}

### Định Dạng SSE Event

Streaming handler của Marketing Agent phát ra bốn loại SSE event mà Web UI render theo thời gian thực:

| Loại Event | Payload | Khi Phát Ra |
|------------|---------|--------------|
| `text` | `{ content: "..." }` | Text chunks từ agent |
| `tool_use` | `{ name: "...", input: {...} }` | Khi một tool invocation bắt đầu |
| `tool_result` | `{ name: "...", status: "...", output: "..." }` | Khi một tool hoàn thành |
| `subagent_progress` | `{ agent: "...", content: "..." }` | Streaming trung gian từ worker agent |

Chat component trong Web UI render mỗi loại khác nhau - text với Markdown, tool use/results dưới dạng panel mở rộng, và subagent progress gắn vào parent tool use block.
