---
title: "Giới Thiệu"
date: 2026-05-29
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

{{% notice tip %}}
**Trước khi bắt đầu**: Bạn cần một tài khoản AWS có quyền triển khai AgentCore Runtimes, Lambda functions, API Gateway, Cognito, DynamoDB, S3, và SSM Parameter Store. Xem phần [Yêu Cầu & Thiết Lập]({{< ref "09-prerequisites" >}}) để biết yêu cầu chi tiết và ước tính chi phí.
{{% /notice %}}

## Thách Thức Marketing Hiện Đại

Để triển khai một chiến dịch marketing, marketer phải điều phối trên ba nền tảng rời rạc:

- **Databricks** - truy vấn phân khúc khách hàng, chạy SQL trên data warehouse, khám phá schema và bảng trong Unity Catalog
- **CleverTap** - quản lý vòng đời chiến dịch: tạo bản nháp, xác thực targeting, xác nhận phân phối qua push, email, SMS, WhatsApp, web push, và webhook
- **TalonOne** - cấu hình khuyến mãi, coupon, chương trình loyalty, phiên khách hàng, và quản lý chiến dịch giảm giá

Mỗi nền tảng yêu cầu thông tin xác thực riêng, hợp đồng API khác nhau, và chuyên môn theo từng lĩnh vực. Marketer dành nhiều thời gian chuyển đổi giữa các công cụ hơn là thiết kế chiến dịch hiệu quả.

{{% notice info %}}
**Chi phí thực tế**: Một chiến dịch đáng lẽ mất vài phút để triển khai kéo dài thành hàng giờ, với việc truyền dữ liệu thủ công dễ gây lỗi giữa các nền tảng.
{{% /notice %}}

## Giải Pháp: Nền Tảng Multi-Agent Điều Khiển Bởi AI

Thay vì buộc marketer tự điều hướng từng nền tảng, chúng tôi giới thiệu **hệ thống multi-agent sử dụng AI** với bốn agent chuyên biệt cộng tác để thực thi chiến dịch từ đầu đến cuối:

- Một **Marketing Agent** điều phối quy trình tạo chiến dịch 3 bước nghiêm ngặt, hướng dẫn người dùng tại mỗi điểm quyết định
- Một **Databricks Agent** khám phá data catalog, chạy SQL, và phân khúc khán giả
- Một **CleverTap Agent** tạo bản nháp chiến dịch, ước tính phạm vi, và xác nhận phân phối sau khi được người dùng phê duyệt
- Một **TalonOne Agent** quản lý khuyến mãi, loyalty, coupon, và phiên khách hàng (bước tùy chọn)

Marketer chỉ cần chat với orchestrator agent bằng ngôn ngữ tự nhiên. Nền tảng xử lý toàn bộ việc điều phối đa hệ thống phía sau.

## Bạn Sẽ Xây Dựng Gì

Đến cuối workshop, bạn sẽ hiểu:

| Thành Phần | Bạn Sẽ Học Được |
|-----------|------------------|
| **Multi-Agent Workflow** | Cách 4 agent phối hợp qua quy trình tạo chiến dịch 3 bước với xác nhận của người dùng ở mỗi giai đoạn |
| **AWS AgentCore** | Cách AgentCore Runtime, Memory, và MCP Gateway loại bỏ độ phức tạp hạ tầng |
| **Platform Agents & MCP** | Cách 3 MCP server Lambda-based cung cấp 25+ tools trên Databricks, CleverTap, và TalonOne |
| **A2A Communication** | Cách agent phân công tác vụ cho nhau với xác thực SigV4, SSE streaming, và lưu trữ phiên |
| **Strands Framework** | Cách open-source Strands Agents cung cấp @tool decorator, hooks, tích hợp MCP, và prompt động |
| **Web UI & Infrastructure** | Cách frontend React/Cloudscape kết nối với 9 Lambda handler, 4 AgentCore endpoint, và hạ tầng CDK |

## AWS AgentCore: Nền Tảng

Toàn bộ hệ thống chạy trên **AWS AgentCore** - dịch vụ được thiết kế chuyên biệt để triển khai và vận hành AI agent ở quy mô lớn. Nó cung cấp ba khả năng cốt lõi:

| Dịch Vụ | Vai Trò Trong Nền Tảng |
|---------|----------------------|
| **AgentCore Runtime** | Host cả 4 agent dưới dạng Docker container với auto-scaling, xác thực SigV4, và observability tích hợp |
| **AgentCore Memory** | Lưu trữ ngữ cảnh hội thoại qua các tin nhắn, cho phép Marketing Agent nhớ các quyết định trước trong chiến dịch dài hạn |
| **AgentCore MCP Gateway** | Định tuyến tool call từ agent đến 3 MCP server Lambda-based với xác thực IAM - tách biệt logic agent khỏi hạ tầng |
