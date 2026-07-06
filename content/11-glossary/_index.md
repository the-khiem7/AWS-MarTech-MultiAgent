---
title: "Glossary"
date: 2026-07-06
weight: 10
chapter: false
pre: " <b> 10. </b> "
---

| Term | Definition |
|------|------------|
| **A2A (Agent-to-Agent)** | Protocol allowing one agent to call another as a remote tool. Used by the Marketing Agent to delegate to worker agents. |
| **AgentCore** | AWS managed service for deploying and operating AI agents at scale, providing Runtime, Memory, and MCP Gateway. |
| **AgentCore Memory** | Persistent conversation context layer that retains information across multiple turns and sessions. |
| **AgentCore Runtime** | Serverless container runtime that hosts AI agent Docker images with built-in health checks, scaling, and SigV4 auth. |
| **Agent ID** | Identifier used by `S3ArtifactHook` to determine the subfolder path (e.g., `orchestrator`, `databricks-agent`). |
| **API Gateway** | AWS service for creating REST APIs. Routes requests to Lambda handlers with Cognito-based authorization. |
| **CDK (Cloud Development Kit)** | Infrastructure-as-code framework for defining AWS resources in TypeScript (or other languages). |
| **Cloudscape** | AWS's open-source design system for building console-quality web applications. |
| **Cognito** | AWS identity service for user authentication, providing OIDC-based login and temporary credential exchange. |
| **DynamoDB** | NoSQL database used to store campaign records with a GSI for active campaign queries. |
| **GSI (Global Secondary Index)** | An index on a DynamoDB table with a different partition key for alternative access patterns. |
| **Gateway MCP Client** | A Strands MCP client that connects to the AgentCore MCP Gateway with SigV4 authentication and tool prefix filtering. |
| **IAM** | AWS Identity and Access Management - controls authentication and authorization for all AWS service calls. |
| **MCP (Model Context Protocol)** | A protocol for exposing tools to AI agents via a standardized interface. The AgentCore MCP Gateway routes MCP calls to Lambda functions. |
| **MCP Gateway** | Managed gateway that routes tool calls from agents to Lambda-based MCP servers with IAM authentication. |
| **MCP Server** | A Lambda function that translates tool calls into a specific platform's native API, using credentials from Secrets Manager. |
| **Rolldown** | JavaScript bundler used to bundle Lambda handler code for deployment. |
| **Secrets Manager** | AWS service for securely storing and rotating credentials used by MCP servers. |
| **SigV4** | AWS Signature Version 4 - request signing protocol used for all agent-to-agent and agent-to-gateway communication. |
| **SSE (Server-Sent Events)** | Streaming protocol used to send real-time events from the Marketing Agent to the Web UI. Four event types: `text`, `tool_use`, `tool_result`, `subagent_progress`. |
| **SSM Parameter Store** | AWS service for storing configuration values, used to hold agent system prompts and model IDs. |
| **Strands Agents** | Open-source Python framework from AWS for building composable AI agents with `@tool` decorators, hooks, and MCP integration. |
| **SubAgentProgress** | SSE event type emitted by a worker agent during streaming, attached to the parent tool use block in the UI. |
| **TanStack Router** | Type-safe file-based routing library for React applications. |
| **tRPC** | Type-safe RPC library for building client-server APIs without REST or GraphQL boilerplate. |
| **Unity Catalog** | Databricks' data governance solution for organizing and discovering data assets across workspaces. |
