---
title: "Multi-Agent Workflow"
date: 2026-05-29
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Marketing Agent thực thi một quy trình tạo chiến dịch ba bước nghiêm ngặt. Người dùng phải xác nhận mỗi bước trước khi tiếp tục - agent từ chối giúp đỡ bất cứ điều gì ngoài workflow này.

## Bước 1 - Xác Định Đối Tượng Mục Tiêu

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as API Handler
    participant Mkt as Marketing Agent
    participant DB as Databricks Agent
    participant GW as MCP Gateway
    participant DBR as Databricks
    UI->>API: Người dùng gửi prompt
    API->>Mkt: Gọi qua AgentCore Runtime
    Mkt->>DB: Phân công tác vụ audience (A2A)
    DB->>GW: Gọi Databricks tools
    GW->>DBR: API calls (SQL, catalog discovery)
    DBR-->>GW: Kết quả
    GW-->>DB: Tool results
    DB-->>Mkt: Dữ liệu audience
    Mkt-->>UI: Stream chi tiết audience (SSE)
    Note over Mkt,UI: Người dùng xem xét và xác nhận audience
{{< /mermaid >}}

**Databricks Agent** xử lý bước này bằng 8 MCP tools:

| Tool | Mục Đích |
|------|---------|
| `list_warehouses` | Khám phá SQL warehouses khả dụng |
| `list_schemas` | Khám phá Unity Catalog schemas |
| `list_tables` | Liệt kê bảng trong schema |
| `get_table` | Kiểm tra tên và kiểu cột |
| `execute_sql` | Chạy truy vấn SQL trên warehouse |
| `get_statement_result` | Poll kết quả truy vấn dài hạn |
| `run_job` | Kích hoạt Databricks job |
| `get_job_run` | Kiểm tra trạng thái job run |

**Workflow**: Agent khám phá nguồn dữ liệu khả dụng, xây dựng và thực thi truy vấn SQL, và trình bày phân khúc khán giả. Kết quả SQL vượt 20 dòng hoặc 10KB được upload lên S3 dạng JSON - agent thông báo cho người dùng vị trí tải về.

{{% notice info %}}
Đây là bước tốn nhiều hội thoại nhất. Một phiên thực tế đã ghi nhận 26 tin nhắn giữa người dùng, Marketing Agent, và Databricks Agent trước khi đạt được xác nhận audience.
{{% /notice %}}

## Bước 2 - Tạo Chiến Dịch Trong CleverTap

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as API Handler
    participant Mkt as Marketing Agent
    participant CT as CleverTap Agent
    participant GW as MCP Gateway
    participant CTR as CleverTap
    UI->>API: Người dùng xác nhận audience
    API->>Mkt: Gọi qua AgentCore Runtime
    Mkt->>CT: Phân công tạo chiến dịch (A2A)
    CT->>GW: Gọi CleverTap tools
    GW->>CTR: API calls (draft, confirm)
    CTR-->>GW: Kết quả
    GW-->>CT: Tool results
    CT-->>Mkt: Chi tiết chiến dịch
    Mkt-->>UI: Stream xác nhận chiến dịch (SSE)
{{< /mermaid >}}

**CleverTap Agent** thực thi workflow draft-first:

| Tool | Mô Tả |
|------|-------------|
| `create_draft_campaign` | Xác thực targeting với `estimate_only=true`; trả về phạm vi ước tính |
| `confirm_draft_campaign` | Thực sự tạo chiến dịch với `estimate_only=false` |
| `list_draft_campaigns` | Liệt kê chiến dịch trong khoảng ngày |
| `get_draft_campaign` | Xem chi tiết một bản nháp cụ thể |
| `update_draft_campaign` | Xác thực lại bản nháp với tham số cập nhật |
| `discard_draft_campaign` | Xóa vĩnh viễn một bản nháp |

**Kênh hỗ trợ**: push, email, SMS, web push, WhatsApp, webhook. Với email, SMS, và WhatsApp, cần có `provider_nick_name`.

{{% notice warning %}}
Agent **luôn** tạo bản nháp trước. Nó không bao giờ tạo chiến dịch trực tiếp nếu chưa có xác nhận của người dùng về phạm vi ước tính.
{{% /notice %}}

## Bước 3 - Tạo Khuyến Mãi Trong TalonOne (Tùy Chọn)

{{< mermaid >}}
sequenceDiagram
    participant UI as Web UI
    participant API as API Handler
    participant Mkt as Marketing Agent
    participant TO as TalonOne Agent
    participant GW as MCP Gateway
    participant TOR as TalonOne
    UI->>API: Người dùng chọn tạo khuyến mãi
    API->>Mkt: Gọi qua AgentCore Runtime
    Mkt->>TO: Phân công tác vụ khuyến mãi (A2A)
    TO->>GW: Gọi TalonOne tools
    GW->>TOR: API calls (campaigns, coupons)
    TOR-->>GW: Kết quả
    GW-->>TO: Tool results
    TO-->>Mkt: Chi tiết khuyến mãi
    Mkt-->>UI: Stream xác nhận khuyến mãi (SSE)
{{< /mermaid >}}

**TalonOne Agent** cung cấp 11 tools trên bốn lĩnh vực:

| Lĩnh Vực | Tools |
|--------|-------|
| **Campaigns** | `list_campaigns`, `get_campaign`, `create_campaign` |
| **Loyalty** | `get_loyalty_program`, `get_customer_loyalty`, `redeem_points` |
| **Coupons** | `list_coupons`, `validate_coupon`, `create_coupon` |
| **Sessions** | `get_customer_session`, `update_customer_session` |

Campaigns mặc định ở trạng thái `disabled` để xem xét trước khi kích hoạt. Agent khám phá campaigns khả dụng bằng `list_campaigns` trước khi thực hiện bất kỳ thao tác nào khác.

## Phía Sau Hậu Trường

Trong khi người dùng thấy một giao diện chat đơn giản, ba cơ chế quan trọng chạy ngầm:

1. **AgentCore Memory** giữ lại ngữ cảnh hội thoại giữa các tin nhắn, để Marketing Agent nhớ người dùng đang ở bước nào và những gì đã được xác nhận trước đó.

2. **Session ID Propagation** - cùng một session ID chảy từ Web UI → Put Chat Lambda → Marketing Agent → A2A calls → Worker Agents → S3. Cả bốn agent ghi vào cùng một thư mục S3.

3. **SSE Streaming** - Marketing Agent phát ra bốn loại event (`text`, `tool_use`, `tool_result`, `subagent_progress`) mà Web UI render theo thời gian thực, cho người dùng thấy được các tool invocation và suy luận trung gian của agent.
