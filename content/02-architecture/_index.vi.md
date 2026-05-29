---
title: "Tổng Quan Kiến Trúc"
date: 2026-05-29
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Tổng Quan Kiến Trúc

## Kiến Trúc Hệ Thống

Nền tảng MarTech multi-agent áp dụng kiến trúc phân lớp, nơi các AI agent chuyên biệt tương tác với các nền tảng marketing thông qua một lớp điều phối thống nhất.

![System Architecture](_static/architecture.jpg)

## Topology Agent

Bốn agent cộng tác để thực thi chiến dịch marketing:

| Agent | Trách Nhiệm | Nền Tảng | Framework |
|-------|---------------|----------|-----------|
| **Marketer Agent** | Điều phối — nhận ý định người dùng, phân công cho sub-agent | AWS AgentCore Runtime | Strands Agents |
| **Databricks Agent** | Phân khúc khách hàng, truy vấn SQL, phân tích dữ liệu | Databricks Unity Catalog + Warehouses | Strands Agents + MCP |
| **CleverTap Agent** | Tạo chiến dịch, gửi thông báo, theo dõi người dùng | CleverTap | Strands Agents + MCP |
| **TalonOne Agent** | Khuyến mãi, coupon, chương trình loyalty, phiên khách hàng | TalonOne | Strands Agents + MCP |

## Luồng Dữ Liệu

Khi marketer mô tả chiến dịch bằng ngôn ngữ tự nhiên:

1. **Marketer Agent** nhận yêu cầu và xác định sub-agent nào cần thiết
2. **Databricks Agent** truy vấn phân khúc khách hàng và trả về hồ sơ khách hàng mục tiêu
3. **CleverTap Agent** tạo chiến dịch với đối tượng đã xác định
4. **TalonOne Agent** gắn khuyến mãi và giảm giá phù hợp
5. Kết quả được tổng hợp về **Marketer Agent**, agent này báo cáo cho người dùng

## Mô Hình Giao Tiếp

Hai mô hình giao tiếp được sử dụng:

{{< tabs >}}

{{< tab name="MCP Gateway" >}}
Mỗi platform agent (Databricks, CleverTap, TalonOne) công bố khả năng của mình dưới dạng **MCP tools** thông qua **AgentCore MCP Gateway**. Agent gọi MCP tools trực tiếp, và gateway xử lý:
- Xác thực IAM SigV4
- Định tuyến đến Lambda-based MCP server tương ứng
- Tách biệt logic agent với hạ tầng
{{< /tab >}}

{{< tab name="A2A (Agent-to-Agent)" >}}
Marketer Agent giao tiếp với các platform agent qua **A2A**, coi mỗi sub-agent như một remote tool. A2A cung cấp:
- Streaming xác thực SigV4 qua SSE events
- Truyền Session ID qua chuỗi tool call
- Báo cáo tiến độ về orchestrator
{{< /tab >}}

{{< /tabs >}}

## Frontend

Giao diện web React xây dựng trên **Cloudscape Design System** — thiết kế theo ngôn ngữ AWS — cung cấp:
- Bảng điều khiển quản lý chiến dịch
- Giao diện chat thời gian thực với Marketer Agent
- Xác thực Cognito và quản lý người dùng

Frontend giao tiếp với backend TypeScript Lambda qua REST APIs (API Gateway), chuyển tiếp yêu cầu đến agent runtime.
