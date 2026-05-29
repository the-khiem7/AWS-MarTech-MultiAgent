---
title: "Giới Thiệu"
date: 2026-05-29
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

# Giới Thiệu

## Thách Thức Marketing Hiện Đại

Các đội ngũ marketing ngày nay hoạt động trên nhiều nền tảng chuyên biệt. Để triển khai một chiến dịch, marketer phải:

- **Truy vấn phân khúc khách hàng** qua kho dữ liệu Databricks
- **Quản lý quy trình chiến dịch** trong CleverTap
- **Cấu hình khuyến mãi và giảm giá** trong TalonOne

Mỗi nền tảng yêu cầu thông tin xác thực riêng, giao diện khác nhau, và chuyên môn theo từng lĩnh vực. Việc chuyển đổi giữa chúng làm chậm quá trình triển khai chiến dịch, tăng tỉ lệ lỗi, và phân mảnh quy trình làm việc.

{{% notice info %}}
**Kết luận**: Marketer dành nhiều thời gian để thao tác giữa các công cụ hơn là thiết kế và thực thi chiến dịch hiệu quả.
{{% /notice %}}

## Giải Pháp: AI-Guided Workflow

Thay vì buộc marketer phải tự điều hướng từng nền tảng, chúng tôi giới thiệu một **hệ thống multi-agent điều khiển bởi AI**:

- **Databricks Agent** xử lý phân khúc khách hàng và truy vấn dữ liệu
- **CleverTap Agent** điều phối tạo và phân phối chiến dịch
- **TalonOne Agent** quản lý khuyến mãi, coupon, và quy tắc loyalty
- **Marketer Agent** (điều phối viên) phối hợp giữa các agent

Marketer chỉ cần mô tả mục tiêu chiến dịch bằng ngôn ngữ tự nhiên, và các agent sẽ cộng tác thực thi trên các nền tảng.

## AWS AgentCore: Nền Tảng

Toàn bộ hệ thống chạy trên **AWS AgentCore** — dịch vụ được thiết kế chuyên biệt để triển khai và vận hành AI agent ở quy mô lớn. Trong các phần tiếp theo, bạn sẽ tìm hiểu từng thành phần:

| Thành Phần | Vai Trò |
|-----------|------|
| **AgentCore Runtime** | Serverless container hosting cho AI agent |
| **AgentCore Memory** | Bộ nhớ phiên liên tục qua các cuộc hội thoại |
| **AgentCore MCP Gateway** | Cổng tool xác thực IAM cho Lambda-based tools |
| **A2A (Agent-to-Agent)** | Giao thức giao tiếp an toàn giữa các agent |

Đến cuối workshop, bạn sẽ hiểu cách xây dựng, triển khai, và điều phối một nền tảng marketing multi-agent cấp production sử dụng AWS AgentCore.
