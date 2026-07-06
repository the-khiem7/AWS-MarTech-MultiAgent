---
title: "Bài Tập Thực Hành"
date: 2026-07-06
weight: 9
chapter: false
pre: " <b> 9. </b> "
---

Thực hành những gì bạn đã học với các bài tập sau. Hoàn thành theo thứ tự — mỗi bài tập xây dựng trên bài trước.

---

## Bài Tập 1: Triển Khai Stack

**Mục tiêu**: Triển khai nền tảng MarTech lên tài khoản AWS và xác minh tất cả tài nguyên được tạo.

**Các bước**:

1. Clone repository và cài đặt dependencies:

```bash
git clone <repo-url>
cd aws-martech-multiagent
pnpm install
uv sync
```

2. Cấu hình credentials bên thứ ba trong `packages/infra/config/default.yaml`. Cung cấp ít nhất credentials cho một platform:

```yaml
deploymentConfig:
  mcp:
    databricks:
      url: "https://your-workspace.cloud.databricks.com"
      token: "dapi-your-personal-access-token"
```

3. Build tất cả packages:

```bash
pnpm run build:all
```

4. Triển khai stack:

```bash
pnpm exec nx deploy @play-c463-z26-rzy-mar-tech/infra "stack-name/*"
```

5. **Xác minh** trong AWS Console:
   - CloudFormation → Stacks → Trạng thái `CREATE_COMPLETE`
   - Bedrock → AgentCore → Runtimes → 4 runtimes tồn tại
   - Bedrock → AgentCore → Gateway → `marketer-gateway` hoạt động với 3 targets
   - Systems Manager → Parameter Store → Parameters cho mỗi agent

---

## Bài Tập 2: Chat với Marketing Agent

**Mục tiêu**: Sử dụng Web UI để tạo chiến dịch và quan sát multi-agent workflow.

**Các bước**:

1. Tìm URL Web UI:
   - CloudFormation → Outputs → `WebUiUrl`

2. **Đăng nhập** bằng admin credentials từ `default.yaml`

3. Mở **Developer Tools** (F12) → **Network** → Lọc `chat`

4. Click **Create Campaign** và đặt tên

5. Trong chat, nhập prompt mô tả đối tượng mục tiêu:

   ```
   Tìm tất cả người dùng premium ở khu vực San Francisco đã mua hàng trong 90 ngày qua. Ước tính kích thước đối tượng.
   ```

6. **Quan sát**:
   - Streaming response trong chat UI
   - SSE events (`data: {"type": "text"...}`)
   - `subagent_progress` khi Databricks Agent làm việc
   - `tool_use` / `tool_result` panels

7. Hoàn thành workflow qua Bước 2 (và tùy chọn Bước 3).

---

## Bài Tập 3: Chỉnh System Prompt

**Mục tiêu**: Thay đổi hành vi của Databricks Agent bằng cách sửa system prompt, không cần deploy lại.

**Các bước**:

1. Vào Web UI → **Configuration** (`/configuration`)

2. Chọn **Databricks Agent** tab

3. Xem system prompt hiện tại

4. **Thêm** hướng dẫn mới:

   ```
   Luôn giới hạn SQL queries tối đa 100 dòng trừ khi người dùng yêu cầu cụ thể nhiều hơn.
   ```

5. Click **Save**

6. Quay lại chat và gửi:

   ```
   Hiển thị tất cả người dùng trong database.
   ```

7. **Xác nhận**: Agent giới hạn kết quả ở 100 dòng.

8. **Khôi phục**: Đổi prompt lại nội dung gốc.

---

## Bài Tập 4: Xem S3 Session Artifacts

**Mục tiêu**: Duyệt S3 session bucket để hiểu audit trail của multi-agent system.

**Các bước**:

1. Tìm sessions bucket name trong CloudFormation Outputs

2. Vào thư mục session (ví dụ: `session-<uuid>/`)

3. **Xem cấu trúc**:

```
session-<uuid>/
├── orchestrator/agent.json
├── orchestrator/messages/message_0.json
├── databricks-agent/agent.json
├── databricks-agent/messages/message_0.json
└── clevertap-agent/agent.json
└── clevertap-agent/messages/message_0.json
```

4. Mở `agent.json` và `message_*.json` để xem cấu trúc

5. So sánh thư mục `orchestrator` và `databricks-agent`

---

## Bài Tập 5: MCP Gateway Tool Filtering

**Mục tiêu**: Hiểu cách MCP Gateway lọc tools để mỗi agent chỉ thấy tools của mình.

**Các bước**:

1. Trong CloudWatch Logs của Databricks Agent runtime, tìm tool invocation entries

2. Ghi nhận tool names có prefix: `databricks-target___execute_sql`

3. Trong Web UI chat, xem `tool_use` SSE event — prefix đã được loại bỏ

4. Kiểm tra TalonOne agent logs — xác nhận nó không thấy `databricks-target___*` tools

---

## Bài Tập Bonus: Mở Rộng Nền Tảng (Lý Thuyết)

**Mục tiêu**: Thiết kế cách thêm agent thứ năm (ví dụ: Salesforce CRM Agent).

**Nhiệm vụ**:

1. **MCP Server**: Mô tả Lambda function và tools (ví dụ: `list_contacts`, `create_lead`)

2. **Gateway Target**: Cần thêm gì vào `GatewayConstruct`?

3. **Worker Agent**: Làm thế nào để tạo agent mới dùng `create_a2a_app` factory?

4. **Orchestrator tool**: Cần thay đổi gì trong Marketing Agent?

5. **IAM**: Cần thêm permissions gì?

---

## Xử Lý Sự Cố

| Triệu Chứng | Nguyên Nhân | Giải Pháp |
|---------|-------------|----------|
| Agent không phản hồi | Thiếu Bedrock model access | Xác minh trong Bedrock console |
| MCP tool trả về 403 | Secrets sai | Kiểm tra Secrets Manager |
| Web UI trắng | Runtime config chưa load | Chạy `load:runtime-config` |
| A2A thất bại | Thiếu IAM permission | Kiểm tra role cho `InvokeAgentRuntime` |
| Campaign không hiện | DynamoDB GSI chưa có | Kiểm tra `CampaignActiveIndex` |
