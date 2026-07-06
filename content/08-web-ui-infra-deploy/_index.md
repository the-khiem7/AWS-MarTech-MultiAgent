---
title: "Web UI & Infrastructure"
date: 2026-05-29
weight: 8
chapter: false
pre: " <b> 8. </b> "
---

## Web UI

The Web UI is a React/TypeScript single-page application that provides the user-facing interface for campaign management and agent interaction. It uses the **Cloudscape Design System** - AWS's open-source design language - for a console-quality experience.

### Technology Stack

| Technology | Purpose |
|-----------|---------|
| **React 19** | UI framework |
| **TanStack Router** | Type-safe file-based routing |
| **TanStack React Query + tRPC** | Server state management and caching |
| **Cloudscape Components** | AWS-styled UI components |
| **Cloudscape Chat Components** | Real-time chat widgets |
| **Cloudscape Board Components** | Dashboard layout |
| **Tailwind CSS 4** | Utility-first styling |
| **Cognito + OIDC** | Authentication via `react-oidc-context` |
| **aws4fetch** | SigV4 request signing |
| **Vite + Rolldown** | Frontend bundling (Vite) + Lambda bundling (Rolldown) |

### Provider Hierarchy

The application wraps all routes in a provider hierarchy:

```
I18nProvider → RuntimeConfigProvider → CognitoAuth → QueryClientProvider → ApiClientProvider
```

### Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | Redirect | Redirects to campaign list |
| `/campaign` | CampaignsList | Paginated campaign table with create modal |
| `/campaign/:id` | CampaignDetail | Campaign details with embedded chat interface |
| `/configuration` | Configuration | Model selector and system prompt editor for all 4 agents |

### Chat Interface

The core interaction component ([`src/components/Chat/`](../../packages/web-ui/src/components/Chat/)):
- Loads conversation history from `GET /chat/:sessionId` on mount
- Sends messages via `PUT /chat` and processes the SSE stream in real-time
- Renders four content block types:
  - **Text** - Markdown via `react-markdown` + `remark-gfm`
  - **Tool use** - Expandable panel showing tool name and input
  - **Tool result** - Expandable panel showing tool output
  - **Subagent progress** - Attached to parent tool use, shows worker agent streaming
- Uses Cloudscape `ChatBubble`, `Avatar`, `LoadingBar`, `PromptInput` components

### API Client

The shared API client (`packages/api/src/client/index.ts`) uses `aws4fetch` for SigV4-signed requests:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `campaign.get(id)` | GET /campaign/:id | Single campaign |
| `campaign.list({ pageSize, nextToken })` | GET /campaign | Paginated list (DynamoDB GSI) |
| `campaign.create({ name })` | POST /campaign | Create with UUID |
| `chat.put({ sessionId, prompt }, onChunk)` | PUT /chat | Streaming via SSE |
| `chat.getHistory(sessionId)` | GET /chat/:sessionId | From AgentCore Memory |
| `configuration.get(agentName)` | GET /configuration/:agentName | From SSM |
| `configuration.put(agentName, config)` | PUT /configuration/:agentName | To SSM |
| `configuration.listModels()` | GET /configuration/models | Bedrock models |
| `sqlResult.get(key)` | GET /sql-result/:key | Presigned S3 URL |

### Authentication

The `CognitoAuth` component wraps the app with `react-oidc-context`. Credentials from the Cognito Identity Pool are exchanged for temporary IAM credentials, which `useSigV4` uses to sign all API requests with a 30-second expiry grace period.

---

## Infrastructure (AWS CDK)

The entire stack is provisioned via AWS CDK (TypeScript) and deployed as a single CloudFormation stack. All constructs are in `packages/infra/`.

### Deployment Configuration

Environment-specific values are configured in `packages/infra/config/default.yaml`:

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

Only the integrations you use need to be configured. Unconfigured ones are safely ignored.

### ApplicationStack Constructs

The `ApplicationStack` instantiates 7 constructs in order:

| Order | Construct | Key Resources |
|-------|-----------|---------------|
| 1 | **UserIdentity** | Cognito User Pool, Identity Pool |
| 2 | **StorageAndData** | Campaigns DynamoDB table, 3 S3 buckets (sessions, SQL results, access logs) |
| 3 | **GatewayConstruct** | AgentCore MCP Gateway, 3 Lambda targets (Databricks, CleverTap, TalonOne) |
| 4 | **AgentConstruct** | 4 AgentCore Runtimes, shared Memory, IAM execution roles |
| 5 | **SeedConfig** | Default agent configs in SSM Parameter Store |
| 6 | **APIConstruct** | API Gateway, 9 Lambda handlers |
| 7 | **WebUi** | S3 + CloudFront static site hosting |

### StorageAndData

- **Campaigns Table** - Partition key `id` (String), PAY_PER_REQUEST, point-in-time recovery. GSI `CampaignActiveIndex` partitioned by `active`, sorted by `createdAt`.
- **Sessions Bucket** - Stores S3 artifact hooks output. EventBridge enabled, server access logging.
- **SQL Results Bucket** - Stores full SQL result sets from Databricks MCP Server. CORS enabled for GET.
- **Access Logs Bucket** - Server access logs for the other buckets.

All buckets enforce SSL and block public access.

### APIConstruct - 9 Lambda Handlers

| Handler | Route | Timeout | IAM Permissions |
|---------|-------|---------|-----------------|
| `getCampaign` | GET /campaign/:id | 30s | DynamoDB GetItem |
| `getCampaigns` | GET /campaign | 30s | DynamoDB Query (GSI) |
| `createCampaign` | POST /campaign | 30s | DynamoDB PutItem |
| `putChat` | PUT /chat | **15 min** | AgentCore Runtime Invoke |
| `getChatHistory` | GET /chat/:sessionId | 30s | AgentCore Memory ListEvents |
| `getAgentConfig` | GET /configuration/:agentName | 30s | SSM GetParameter |
| `putAgentConfig` | PUT /configuration/:agentName | 30s | SSM PutParameter |
| `listBedrockModels` | GET /configuration/models | 30s | Bedrock ListFoundationModels, ListInferenceProfiles |
| `getSqlResult` | GET /sql-result/:key+ | 30s | S3 GetObject |

All handlers use Node.js 22.x runtime with X-Ray tracing enabled (except `putChat` which uses response streaming). Lambda handlers are bundled with Rolldown.

{{% notice warning %}}
**Preview service**: The `@aws-cdk/aws-bedrock-agentcore-alpha` CDK construct is an alpha/preview package. APIs and behaviors may change before general availability.
{{% /notice %}}

### Chat History Consolidation

The `getChatHistory` handler does significant post-processing:
- Parses conversational payload events from AgentCore Memory
- Extracts text, tool use, and tool result blocks
- Strips gateway prefixes from tool names (`target___toolname` → `toolname`)
- Consolidates consecutive same-role messages
- Merges tool result messages into the preceding assistant message

This ensures the UI displays a clean, coherent conversation despite the internal complexity of multi-agent orchestration.

---

## Build & Deploy

### Monorepo Structure

```
packages/
  web-ui/          # React frontend (Vite, Cloudscape, TanStack)
  api/             # Lambda handlers (Node.js, bundled with Rolldown)
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

### Deployment Workflow

```bash
# 1. Install dependencies
pnpm install
uv sync

# 2. Build all packages
pnpm run build:all

# 3. Deploy to AWS
pnpm exec nx deploy @play-c463-z26-rzy-mar-tech/infra "stack-name/*"
```

### Local Development

After initial deployment, run the UI locally:

```bash
# Load runtime config from deployed stack
pnpm exec nx run @play-c463-z26-rzy-mar-tech/web-ui:load:runtime-config

# Start local dev server with HMR
pnpm exec nx serve @play-c463-z26-rzy-mar-tech/web-ui
```

{{% notice tip %}}
The system prompt for each agent is configurable via the Web UI Configuration page or SSM Parameter Store, allowing you to tune agent behavior without redeploying code. See [Strands Agents Framework]({{< ref "07-strands-framework" >}}) for details.
{{% /notice %}}
