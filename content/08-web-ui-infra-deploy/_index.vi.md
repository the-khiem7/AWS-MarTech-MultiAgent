---
title: "Web UI & Hạ Tầng"
date: 2026-05-29
weight: 8
chapter: false
pre: " <b> 8. </b> "
---

## Web UI

Web UI là một single-page application React/TypeScript cung cấp giao diện người dùng cho quản lý chiến dịch và tương tác agent. Nó sử dụng **Cloudscape Design System** - design language mã nguồn mở của AWS - cho trải nghiệm chất lượng console.

### Technology Stack

| Công Nghệ | Mục Đích |
|-----------|---------|
| **React 19** | UI framework |
| **TanStack Router** | Type-safe file-based routing |
| **TanStack React Query + tRPC** | Server state management và caching |
| **Cloudscape Components** | UI components phong cách AWS |
| **Cloudscape Chat Components** | Widget chat thời gian thực |
| **Cloudscape Board Components** | Bố cục dashboard |
| **Tailwind CSS 4** | Utility-first styling |
| **Cognito + OIDC** | Xác thực qua `react-oidc-context` |
| **aws4fetch** | Ký request SigV4 |
| **Vite + Rolldown** | Bundling frontend (Vite) + bundling Lambda (Rolldown) |

### Provider Hierarchy

Application bọc tất cả routes trong một provider hierarchy:

```
I18nProvider → RuntimeConfigProvider → CognitoAuth → QueryClientProvider → ApiClientProvider
```

### Routes

| Route | Component | Mô Tả |
|-------|-----------|-------------|
| `/` | Redirect | Chuyển hướng đến danh sách chiến dịch |
| `/campaign` | CampaignsList | Bảng chiến dịch phân trang với modal tạo mới |
| `/campaign/:id` | CampaignDetail | Chi tiết chiến dịch với chat interface nhúng |
| `/configuration` | Configuration | Chọn model và sửa system prompt cho cả 4 agent |

### Giao Diện Chat

Core interaction component ([`src/components/Chat/`](../../packages/web-ui/src/components/Chat/)):
- Tải lịch sử hội thoại từ `GET /chat/:sessionId` khi mount
- Gửi tin nhắn qua `PUT /chat` và xử lý SSE stream theo thời gian thực
- Render bốn loại content block:
  - **Text** - Markdown qua `react-markdown` + `remark-gfm`
  - **Tool use** - Panel mở rộng hiển thị tên tool và input
  - **Tool result** - Panel mở rộng hiển thị output của tool
  - **Subagent progress** - Gắn vào parent tool use, hiển thị streaming từ worker agent
- Sử dụng Cloudscape `ChatBubble`, `Avatar`, `LoadingBar`, `PromptInput`

### API Client

Shared API client (`packages/api/src/client/index.ts`) dùng `aws4fetch` cho SigV4-signed requests:

| Method | Endpoint | Mô Tả |
|--------|----------|-------------|
| `campaign.get(id)` | GET /campaign/:id | Chiến dịch đơn |
| `campaign.list({ pageSize, nextToken })` | GET /campaign | Danh sách phân trang (DynamoDB GSI) |
| `campaign.create({ name })` | POST /campaign | Tạo với UUID |
| `chat.put({ sessionId, prompt }, onChunk)` | PUT /chat | Streaming qua SSE |
| `chat.getHistory(sessionId)` | GET /chat/:sessionId | Từ AgentCore Memory |
| `configuration.get(agentName)` | GET /configuration/:agentName | Từ SSM |
| `configuration.put(agentName, config)` | PUT /configuration/:agentName | Đến SSM |
| `configuration.listModels()` | GET /configuration/models | Bedrock models |
| `sqlResult.get(key)` | GET /sql-result/:key | Presigned S3 URL |

### Xác Thực

`CognitoAuth` component bọc app với `react-oidc-context`. Credentials từ Cognito Identity Pool được trao đổi thành temporary IAM credentials, được `useSigV4` dùng để ký tất cả API request với grace period 30 giây.

---

## Hạ Tầng (AWS CDK)

Toàn bộ stack được triển khai qua AWS CDK (TypeScript) dưới dạng một CloudFormation stack. Tất cả constructs nằm trong `packages/infra/`.

### Cấu Hình Triển Khai

Giá trị theo môi trường được cấu hình trong `packages/infra/config/default.yaml`:

```yaml
deploymentConfig:
  adminUser:
    email: admin@example.com
  mcp:
    databricks:
      url: ""
      token: ""
    clevertap:
      projectId: ""
      passcode: ""
      region: ""
    talonone:
      baseUrl: ""
      applicationId: ""
      managementKey: ""
      integrationKey: ""
```

Chỉ những integration bạn sử dụng mới cần cấu hình. Những cái chưa cấu hình sẽ được bỏ qua an toàn.

### ApplicationStack Constructs

`ApplicationStack` khởi tạo 7 constructs theo thứ tự:

| Thứ Tự | Construct | Tài Nguyên Chính |
|-------|-----------|---------------|
| 1 | **UserIdentity** | Cognito User Pool, Identity Pool |
| 2 | **StorageAndData** | Campaigns DynamoDB table, 3 S3 buckets (sessions, SQL results, access logs) |
| 3 | **GatewayConstruct** | AgentCore MCP Gateway, 3 Lambda targets (Databricks, CleverTap, TalonOne) |
| 4 | **AgentConstruct** | 4 AgentCore Runtimes, Memory dùng chung, IAM execution roles |
| 5 | **SeedConfig** | Default agent configs trong SSM Parameter Store |
| 6 | **APIConstruct** | API Gateway, 9 Lambda handlers |
| 7 | **WebUi** | S3 + CloudFront static site hosting |

### StorageAndData

- **Campaigns Table** - Partition key `id` (String), PAY_PER_REQUEST, point-in-time recovery. GSI `CampaignActiveIndex` phân vùng bởi `active`, sắp xếp bởi `createdAt`.
- **Sessions Bucket** - Lưu output của S3 artifact hooks. EventBridge enabled, server access logging.
- **SQL Results Bucket** - Lưu bộ SQL results đầy đủ từ Databricks MCP Server. CORS enabled cho GET.
- **Access Logs Bucket** - Server access logs cho các bucket khác.

Tất cả buckets bắt buộc SSL và chặn public access.

### APIConstruct - 9 Lambda Handlers

| Handler | Route | Timeout | IAM Permissions |
|---------|-------|---------|-----------------|
| `getCampaign` | GET /campaign/:id | 30s | DynamoDB GetItem |
| `getCampaigns` | GET /campaign | 30s | DynamoDB Query (GSI) |
| `createCampaign` | POST /campaign | 30s | DynamoDB PutItem |
| `putChat` | PUT /chat | **15 phút** | AgentCore Runtime Invoke |
| `getChatHistory` | GET /chat/:sessionId | 30s | AgentCore Memory ListEvents |
| `getAgentConfig` | GET /configuration/:agentName | 30s | SSM GetParameter |
| `putAgentConfig` | PUT /configuration/:agentName | 30s | SSM PutParameter |
| `listBedrockModels` | GET /configuration/models | 30s | Bedrock ListFoundationModels, ListInferenceProfiles |
| `getSqlResult` | GET /sql-result/:key+ | 30s | S3 GetObject |

Tất cả handlers dùng Node.js 22.x runtime với X-Ray tracing (trừ `putChat` dùng response streaming). Lambda handlers được bundle bằng Rolldown.

{{% notice warning %}}
**Dịch vụ xem trước**: CDK construct `@aws-cdk/aws-bedrock-agentcore-alpha` là gói alpha/preview. API và hành vi có thể thay đổi trước khi GA.
{{% /notice %}}

### Chat History Consolidation

Handler `getChatHistory` thực hiện xử lý đáng kể:
- Phân tích conversational payload events từ AgentCore Memory
- Trích xuất text, tool use, và tool result blocks
- Loại bỏ gateway prefixes khỏi tên tool (`target___toolname` → `toolname`)
- Hợp nhất các tin nhắn cùng role liên tiếp
- Gộp tool result vào assistant message phía trước

Điều này đảm bảo UI hiển thị hội thoại sạch và mạch lạc bất chấp độ phức tạp nội bộ của multi-agent orchestration.

---

## Build & Triển Khai

### Cấu Trúc Monorepo

```
packages/
  web-ui/          # React frontend (Vite, Cloudscape, TanStack)
  api/             # Lambda handlers (Node.js, bundled với Rolldown)
  infra/           # AWS CDK (TypeScript)
  common/
    constructs/    # Shared CDK constructs
    types/         # Deployment config types
  agents/
    common/        # Shared Python agent utilities
    marketer/      # Marketing Agent (Python)
    databricks/    # Databricks Agent (Python)
    clevertap/     # CleverTap Agent (Python)
    talonone/      # TalonOne Agent (Python)
```

### Quy Trình Triển Khai

```bash
# 1. Cài đặt dependencies
pnpm install
uv sync

# 2. Build tất cả packages
pnpm run build:all

# 3. Triển khai lên AWS
pnpm exec nx deploy @play-c463-z26-rzy-mar-tech/infra "stack-name/*"
```

### Phát Triển Local

Sau khi triển khai lần đầu, chạy UI locally:

```bash
# Tải runtime config từ stack đã triển khai
pnpm exec nx run @play-c463-z26-rzy-mar-tech/web-ui:load:runtime-config

# Chạy dev server với HMR
pnpm exec nx serve @play-c463-z26-rzy-mar-tech/web-ui
```

{{% notice tip %}}
System prompt cho mỗi agent có thể cấu hình qua trang Configuration của Web UI hoặc SSM Parameter Store, cho phép điều chỉnh hành vi agent mà không cần deploy lại code. Xem [Strands Agents Framework]({{< ref "07-strands-framework" >}}) để biết chi tiết.
{{% /notice %}}
