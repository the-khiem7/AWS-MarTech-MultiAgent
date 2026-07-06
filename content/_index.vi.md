---
title: "Agentic AI cho MarTech"
date: 2026-05-29
weight: 1
chapter: false
---

# Agentic AI cho MarTech

Nền tảng quản lý chiến dịch marketing sử dụng AI, xây dựng trên AWS, cho phép marketer điều phối chiến dịch trên Databricks, CleverTap, và TalonOne thông qua hệ thống multi-agent thông minh.

## Tổng Quan Điều Hành

Các đội marketing phụ thuộc vào một hệ sinh thái các nền tảng chuyên biệt rời rạc. Triển khai một chiến dịch đơn lẻ yêu cầu điều hướng Databricks để phân tích đối tượng, CleverTap để gửi chiến dịch, và TalonOne cho khuyến mãi và loyalty. Mỗi nền tảng đòi hỏi thông tin xác thực riêng, hợp đồng API khác nhau, và chuyên môn theo từng lĩnh vực. Kết quả: quy trình đáng lẽ mất vài phút kéo dài thành hàng giờ, với việc truyền dữ liệu thủ công dễ gây lỗi giữa các hệ thống.

**Agentic AI cho MarTech** giải quyết vấn đề này bằng một hệ thống multi-agent thông minh, hợp nhất toàn bộ quy trình làm việc sau một giao diện chat duy nhất. Bốn AI agent chuyên biệt cộng tác tự động — với sự phê duyệt của con người tại mỗi điểm quyết định — để thực thi chiến dịch từ đầu đến cuối.

## Giá Trị Kinh Doanh

| Chỉ Số | Trước Khi Áp Dụng | Sau Khi Áp Dụng |
|--------|--------|-------|
| Thời gian triển khai chiến dịch | Hàng giờ (chuyển đổi thủ công) | Vài phút (chat có hướng dẫn) |
| Điều phối đa nền tảng | Copy-paste thủ công | Tự động hóa qua A2A |
| Audit trail | Rải rác, không tồn tại | Thống nhất trong S3 mỗi phiên |
| Thay đổi hành vi agent | Cần deploy lại code | Tức thì qua SSM parameter |
| Quản lý credentials | API key riêng từng nền tảng | Tập trung trong AWS Secrets Manager |

## Tổng Quan Giải Pháp

Nền tảng triển khai bốn AI agent trên **AWS AgentCore** — dịch vụ quản lý được thiết kế riêng cho AI agent production:

![Kiến Trúc Hệ Thống](02-architecture/_static/architecture.jpg)

**Marketing Agent** đóng vai trò điều phối thông minh, thực thi quy trình ba bước có cấu trúc:

1. **Khám Phá Đối Tượng** — Databricks Agent khám phá Unity Catalog, chạy SQL, và trình bày phân khúc khán giả để phê duyệt.
2. **Tạo Chiến Dịch** — CleverTap Agent tạo bản nháp với ước tính phạm vi tiếp cận, chỉ xác nhận gửi sau khi người dùng cho phép.
3. **Thiết Lập Khuyến Mãi** *(tùy chọn)* — TalonOne Agent tạo chiến dịch khuyến mãi, quản lý coupon, và xử lý đổi điểm loyalty.

Tại mọi bước, hệ thống chờ xác nhận của con người. Agent không bao giờ hành động tự động nếu chưa có sự đồng ý.

## Nền Tảng Kỹ Thuật

- **Runtime**: 4 container serverless (Marketing, Databricks, CleverTap, TalonOne) với IAM roles riêng biệt và xác thực SigV4
- **Giao tiếp**: Giao thức Agent-to-Agent (A2A) với SSE streaming và session ID lan truyền qua toàn bộ chuỗi gọi
- **Truy cập công cụ**: 25 Model Context Protocol (MCP) tools trên 3 Lambda server, định tuyến qua AgentCore MCP Gateway
- **Frontend**: React 19 với Cloudscape Design System, TanStack Router, và chat thời gian thực qua SSE streaming
- **Hạ tầng**: Một AWS CDK stack (TypeScript) — API Gateway, 9 Lambda handlers, DynamoDB, Cognito, S3, CloudFront
- **Framework Agent**: Strands Agents mã nguồn mở với `@tool` decorator, SSM prompt động, hỗ trợ đa model, và lifecycle hooks

## Tại Sao Điều Này Quan Trọng

Đây không phải là bài tập lý thuyết. Kiến trúc được thiết kế cho sử dụng production thực tế:

- **Không phụ thuộc nền tảng** — Mẫu MCP giúp thêm nền tảng mới (Salesforce, HubSpot, v.v.) chỉ cần viết một Lambda server và Gateway target mới
- **Chi phí hiệu quả** — Ước tính ~$76-186/tháng cho phát triển với dịch vụ trả theo sử dụng
- **Có thể quan sát** — Mọi tin nhắn agent được lưu vào S3 trên cả bốn agent, tạo audit trail hoàn chỉnh
- **Linh hoạt** — System prompt và model selection có thể cấu hình runtime qua SSM Parameter Store, không cần deploy lại code

## Các Bước Tiếp Theo

1. Xem [yêu cầu và ước tính chi phí](09-prerequisites) để chuẩn bị môi trường
2. Làm theo [bài tập thực hành](10-exercises) để triển khai stack và chạy chiến dịch trực tiếp
3. Khám phá [kiến trúc chi tiết](02-architecture) để hiểu sâu về triển khai
