---
title: "Platform Agents & MCP Servers"
date: 2026-05-29
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

Ba platform-specific agent xử lý các thao tác Databricks, CleverTap, và TalonOne. Mỗi agent kết nối với một Lambda-based MCP server chuyên dụng phía sau AgentCore MCP Gateway.

---

## Databricks Agent

**Vị trí**: `packages/agents/databricks/`

**Kiến trúc**: Python FastAPI (qua shared `create_a2a_app` factory) → Strands Agent → MCP Gateway Client (prefix: `databricks-target___`) → Databricks MCP Lambda

**Cấu hình Agent**:

```python
def get_databricks_agent():
    mcp_client = get_gateway_mcp_client("databricks-target")
    config = load_configuration()

    return Agent(
        name="Databricks Agent",
        description="Data analytics agent",
        system_prompt=config.get("systemPrompt"),
        tools=[current_time, mcp_client],
        model=config.get("modelId"),
    )
```

### MCP Server Tools (8)

| Tool | Databricks API | Mô Tả |
|------|---------------|-------------|
| `execute_sql` | POST `/api/2.0/sql/statements` | Thực thi SQL với catalog, schema, row_limit tùy chọn |
| `get_statement_result` | GET `/api/2.0/sql/statements/{id}` | Poll kết quả của statement đang chạy |
| `list_warehouses` | GET `/api/2.0/sql/warehouses` | Liệt kê tất cả SQL warehouses |
| `list_schemas` | GET `/api/2.1/unity-catalog/schemas` | Liệt kê schemas trong catalog |
| `list_tables` | GET `/api/2.1/unity-catalog/tables` | Liệt kê bảng trong schema |
| `get_table` | GET `/api/2.1/unity-catalog/tables/{name}` | Xem chi tiết bảng (cột, kiểu) |
| `run_job` | POST `/api/2.1/jobs/run-now` | Kích hoạt Databricks job |
| `get_job_run` | GET `/api/2.1/jobs/runs/get` | Kiểm tra trạng thái job run |

**Credentials**: Lưu trong Secrets Manager dạng `{ url, token }` (workspace URL + Personal Access Token).

**Cắt Ngắn SQL Results**: Kết quả vượt 20 dòng hoặc 10KB được upload lên S3. Response bao gồm `_truncated.total_rows`, `_truncated.preview_rows`, và `_truncated.full_result_s3_uri`. Web UI cung cấp link tải qua `GET /sql-result/{key}`.

### Hướng Dẫn Workflow

System prompt của agent hướng dẫn:
1. Dùng `list_warehouses` để tìm warehouses khả dụng
2. Dùng `list_schemas` và `list_tables` để khám phá dữ liệu trước khi viết query
3. Dùng `get_table` để hiểu tên và kiểu cột trước khi xây dựng SQL
4. Poll với `get_statement_result` nếu query vẫn đang chạy
5. Thông báo cho người dùng vị trí S3 khi kết quả bị cắt ngắn

---

## CleverTap Agent

**Vị trí**: `packages/agents/clevertap/`

**Kiến trúc**: Python FastAPI (qua shared `create_a2a_app` factory) → Strands Agent → MCP Gateway Client (prefix: `clevertap-target___`) → CleverTap MCP Lambda

### MCP Server Tools (6)

| Tool | CleverTap API | Mô Tả |
|------|--------------|-------------|
| `create_draft_campaign` | POST `/1/targets/create.json` | Xác thực với `estimate_only=true`; trả về phạm vi ước tính |
| `confirm_draft_campaign` | POST `/1/targets/create.json` | Tạo chiến dịch với `estimate_only=false` |
| `list_draft_campaigns` | POST `/1/targets/list.json` | Liệt kê chiến dịch trong khoảng ngày (YYYYMMDD) |
| `get_draft_campaign` | POST `/1/targets/result.json` | Xem chi tiết bản nháp |
| `update_draft_campaign` | POST `/1/targets/create.json` | Xác thực lại bản nháp với tham số cập nhật |
| `discard_draft_campaign` | POST `/1/targets/stop.json` | Xóa vĩnh viễn bản nháp |

**Credentials**: Lưu trong Secrets Manager dạng `{ projectId, passcode, region }`.

**Audience Targeting**: Server dịch `user_property_filters` (mảng `{ name, operator, value }`) thành ngôn ngữ query `common_profile_properties.profile_fields` của CleverTap. Hỗ trợ event-based filters tùy chọn với `event_name`, `from`, và `to`.

**Kênh**: push, email, SMS, webpush, whatsapp, webhook.

### Workflow Draft-First

1. Thu thập thông tin cần thiết: name, channel, content, và `user_property_filters`
2. Luôn `create_draft_campaign` trước - không bao giờ gửi mà không có bản nháp
3. Trình bày phạm vi ước tính và yêu cầu xác nhận
4. Nếu xác nhận → `confirm_draft_campaign`; nếu cần thay đổi → `update_draft_campaign`
5. Nếu hủy → `discard_draft_campaign`

---

## TalonOne Agent

**Vị trí**: `packages/agents/talonone/`

**Kiến trúc**: Python FastAPI (qua shared `create_a2a_app` factory) → Strands Agent → MCP Gateway Client (prefix: `talonone-target___`) → TalonOne MCP Lambda

### MCP Server Tools (11)

| Lĩnh Vực | Tool | API | Mô Tả |
|--------|------|-----|-------------|
| **Campaigns** | `list_campaigns` | Management API | Liệt kê với state filter và phân trang |
| | `get_campaign` | Management API | Xem chi tiết campaign theo ID |
| | `create_campaign` | Management API | Tạo campaign mới (mặc định `disabled`) |
| **Sessions** | `get_customer_session` | Management API | Lấy sessions theo profile ID |
| | `update_customer_session` | Integration API | Cập nhật/tạo session với cart items |
| **Loyalty** | `get_loyalty_program` | Management API | Xem chi tiết chương trình hoặc liệt kê tất cả |
| | `get_customer_loyalty` | Management API | Lấy loyalty ledger balances |
| | `redeem_points` | Management API | Trừ điểm từ balance |
| **Coupons** | `list_coupons` | Management API | Liệt kê coupon cho campaign |
| | `validate_coupon` | Management API | Tìm coupon theo code trên tất cả campaigns |
| | `create_coupon` | Management API | Tạo coupon (code, discount type, value) |

**Credentials**: Lưu trong Secrets Manager dạng `{ baseUrl, applicationId, managementKey, integrationKey }`. Management API dùng auth `ManagementKey-v1`; Integration API dùng auth `ApiKey-v1`.

### Hướng Dẫn Workflow

1. Dùng `list_campaigns` trước các thao tác khác
2. Dùng `get_customer_session` trước khi cập nhật
3. Kiểm tra loyalty với `get_customer_loyalty` trước khi đổi điểm
4. Xác thực coupon với `validate_coupon` trước khi áp dụng
5. Tìm campaign_id qua `list_campaigns` trước khi tạo coupon
6. Campaign mới mặc định `disabled` để xem xét

---

## Shared Agent Utilities

Tất cả agent chia sẻ các Python utility trong `packages/agents/common/common/`:

| Module | Mục Đích |
|--------|---------|
| `a2a_server.py` | `create_a2a_app()` factory - bọc Strands Agent trong A2A Server với FastAPI |
| `gateway.py` | `get_gateway_mcp_client(target_name)` - tạo MCP client với xác thực SigV4 và lọc tool theo prefix |
| `config.py` | `load_configuration()` - đọc agent config (modelId, systemPrompt) từ SSM Parameter Store |
| `s3_artifact.py` | `S3ArtifactHook` - ghi tin nhắn hội thoại vào S3 để audit |

{{% notice tip %}}
Marketing Agent dùng custom FastAPI server với endpoint `/invocations` (yêu cầu bởi AgentCore Runtime) thay vì shared `create_a2a_app` factory, vì nó hoạt động như orchestrator chứ không phải worker.
{{% /notice %}}
