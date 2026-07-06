---
title: "Thuật Ngữ"
date: 2026-07-06
weight: 10
chapter: false
pre: " <b> 10. </b> "
---

| Thuật Ngữ | Định Nghĩa |
|------|------------|
| **A2A (Agent-to-Agent)** | Giao thức cho phép một agent gọi agent khác như tool từ xa. Marketing Agent dùng để ủy thác cho worker agents. |
| **AgentCore** | Dịch vụ quản lý của AWS để triển khai và vận hành AI agent ở quy mô lớn, cung cấp Runtime, Memory, và MCP Gateway. |
| **AgentCore Memory** | Lớp bộ nhớ hội thoại liên tục giữ lại thông tin qua nhiều lượt và phiên. |
| **AgentCore Runtime** | Container runtime serverless host Docker image AI agent với health check, scaling, và xác thực SigV4 tích hợp. |
| **API Gateway** | Dịch vụ AWS tạo REST API. Định tuyến request đến Lambda handlers với xác thực Cognito. |
| **CDK (Cloud Development Kit)** | Framework Infrastructure-as-Code định nghĩa tài nguyên AWS bằng TypeScript. |
| **Cloudscape** | Design system mã nguồn mở của AWS cho ứng dụng web chất lượng console. |
| **Cognito** | Dịch vụ định danh AWS cho xác thực người dùng, cung cấp OIDC login và trao đổi credential tạm thời. |
| **DynamoDB** | NoSQL database lưu campaign records với GSI cho truy vấn chiến dịch đang hoạt động. |
| **MCP Gateway** | Gateway quản lý định tuyến tool call từ agent đến Lambda-based MCP server với xác thực IAM. |
| **MCP Server** | Lambda function dịch tool call thành API native của nền tảng cụ thể, dùng credentials từ Secrets Manager. |
| **SigV4** | AWS Signature Version 4 - giao thức ký request dùng cho mọi giao tiếp agent-to-agent và agent-to-gateway. |
| **SSE (Server-Sent Events)** | Giao thức streaming gửi sự kiện thời gian thực từ Marketing Agent đến Web UI. Bốn loại: `text`, `tool_use`, `tool_result`, `subagent_progress`. |
| **SSM Parameter Store** | Dịch vụ AWS lưu giá trị cấu hình, dùng để chứa system prompt và model ID của agent. |
| **Strands Agents** | Framework Python mã nguồn mở từ AWS để xây dựng AI agent có thể kết hợp với `@tool` decorator, hooks, và tích hợp MCP. |
| **TanStack Router** | Thư viện routing dạng file type-safe cho ứng dụng React. |
| **Unity Catalog** | Giải pháp quản trị dữ liệu của Databricks để tổ chức và khám phá tài nguyên dữ liệu. |
