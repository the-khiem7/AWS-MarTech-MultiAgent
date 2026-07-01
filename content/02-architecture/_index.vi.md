---
title: "Tổng Quan Kiến Trúc"
date: 2026-05-29
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## Kiến Trúc Hệ Thống

Nền tảng MarTech áp dụng kiến trúc phân lớp, nơi bốn AI agent chuyên biệt tương tác với các nền tảng marketing thông qua một lớp điều phối thống nhất.

![System Architecture](_static/architecture.jpg)

## Topology Agent

| Agent | Vai Trò | Khả Năng Chính | Runtime |
|-------|------|-----------------|---------|
| **Marketing Agent** | Điều phối — thực thi workflow 3 bước, thu thập xác nhận người dùng | A2A delegation, SSE streaming, S3 artifact hooks, AgentCore Memory | Custom FastAPI + Strands |
| **Databricks Agent** | Phân tích dữ liệu và phân khúc khán giả | 8 MCP tools (SQL, Unity Catalog, Jobs) | A2A Server (Strands) |
| **CleverTap Agent** | Quản lý vòng đời chiến dịch | 6 MCP tools (draft, confirm, list, update, discard) | A2A Server (Strands) |
| **TalonOne Agent** | Khuyến mãi, coupon, loyalty | 11 MCP tools (campaigns, coupons, loyalty, sessions) | A2A Server (Strands) |

## Technology Stack

{{< mermaid >}}
graph TD
    A[Trình Duyệt] -->|REST + SigV4| B[API Gateway + Cognito]
    B --> C[9 Lambda Handlers]
    C --> D[Marketing Agent<br/>AgentCore Runtime]
    D -->|A2A| E[Databricks Agent]
    D -->|A2A| F[CleverTap Agent]
    D -->|A2A| G[TalonOne Agent]
    E -->|MCP| H[AgentCore MCP Gateway]
    F -->|MCP| H
    G -->|MCP| H
    H --> I[MCP Lambda: Databricks]
    H --> J[MCP Lambda: CleverTap]
    H --> K[MCP Lambda: TalonOne]
    I --> L[Databricks API]
    J --> M[CleverTap API]
    K --> N[TalonOne API]
    D --> O[AgentCore Memory]
    D --> P[S3: Session Artifacts]
    E --> P
    F --> P
    G --> P
{{< /mermaid >}}

## Lớp Hạ Tầng (AWS CDK)

Toàn bộ stack được triển khai qua AWS CDK trong một CloudFormation stack với 7 constructs:

| Construct | Tài Nguyên | Mục Đích |
|-----------|-----------|---------|
| **UserIdentity** | Cognito User Pool + Identity Pool | Xác thực OIDC với trao đổi credential SigV4 |
| **StorageAndData** | DynamoDB table (Campaigns) + 3 S3 buckets | Lưu trữ chiến dịch, session artifacts, SQL results, access logs |
| **GatewayConstruct** | AgentCore MCP Gateway + 3 Lambda targets | Định tuyến tool call với IAM auth; 8+6+11 tools trên 3 targets |
| **AgentConstruct** | 4 AgentCore Runtime + Memory dùng chung | Host tất cả agent với IAM roles cho Bedrock, SSM, và gateway |
| **APIConstruct** | API Gateway + 9 Lambda handlers | REST endpoints cho campaigns, chat, configuration, SQL results |
| **WebUi** | S3 + CloudFront | Static site hosting cho frontend React/Cloudscape |
| **SeedConfig** | SSM Parameter Store entries | Model ID và system prompt mặc định cho cả 4 agent |

## Cách Các Agent Kết Nối

Hai mô hình giao tiếp kết nối bốn agent:

{{< tabs >}}

{{< tab name="MCP Gateway" >}}
Mỗi platform agent (Databricks, CleverTap, TalonOne) truy cập tools qua **AgentCore MCP Gateway**. Gateway:

- Xác thực bằng **IAM SigV4** — không cần quản lý API key
- Định tuyến đến **Lambda-based MCP server** tương ứng dựa trên tên tool
- Lọc tools theo prefix (`databricks-target___execute_sql` → `execute_sql`)
- Cô lập logic agent khỏi thay đổi hạ tầng — cập nhật Lambda mà không cần sửa code agent

Credentials cho mỗi nền tảng bên thứ ba được lưu trong **AWS Secrets Manager** và inject vào môi trường thực thi Lambda.
{{< /tab >}}

{{< tab name="A2A (Agent-to-Agent)" >}}
**Marketing Agent** giao tiếp với platform agent qua **A2A**, coi mỗi sub-agent như một remote tool. A2A cung cấp:

- **Streaming xác thực SigV4** qua SSE events (4 loại event: text, tool_use, tool_result, subagent_progress)
- **Truyền Session ID** — cùng một session chảy qua tất cả agent, cho phép lưu trữ S3 artifact thống nhất và ngữ cảnh AgentCore Memory
- **Báo cáo tiến độ** — cập nhật trung gian từ worker agent stream về UI theo thời gian thực
- **IAM access control** — execution role của Marketing Agent cấp quyền `InvokeAgentRuntime` và `GetAgentCard` cho từng worker agent
{{< /tab >}}

{{< /tabs >}}

## Luồng Dữ Liệu

Khi marketer mô tả mục tiêu chiến dịch trong giao diện chat:

1. **Web UI** gửi prompt đến API Gateway qua PUT request ký SigV4
2. **Put Chat Lambda** gọi Marketing Agent trên AgentCore Runtime với session ID
3. **Marketing Agent** thực thi system prompt 3 bước, phân công cho worker agent qua A2A
4. Mỗi **worker agent** gọi tools qua MCP Gateway, gateway định tuyến đến Lambda tương ứng
5. **MCP Lambda** dịch tool call thành API của nền tảng, lấy credentials từ Secrets Manager, trả về kết quả
6. Kết quả chảy ngược qua A2A streaming events đến Marketing Agent
7. Marketing Agent **stream SSE events** qua Lambda về Web UI theo thời gian thực
8. Tất cả agent ghi tin nhắn hội thoại vào **S3** qua `S3ArtifactHook` để audit và kiểm tra
