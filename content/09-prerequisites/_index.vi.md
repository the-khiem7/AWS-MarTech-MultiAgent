---
title: "Yêu Cầu & Thiết Lập"
date: 2026-07-06
weight: -1
chapter: false
pre: " <b> 0. </b> "
---

Trước khi triển khai nền tảng MarTech multi-agent, hãy đảm bảo môi trường và tài khoản AWS của bạn được cấu hình đúng.

## Yêu Cầu Tài Khoản AWS

| Tài Nguyên | Yêu Cầu Tối Thiểu |
|----------|-----------------|
| Tài khoản AWS | Tài khoản hoạt động, đã bật billing |
| IAM Permissions | Admin-level hoặc các quyền cụ thể bên dưới |
| Service Quotas | Bedrock model access, Lambda concurrency (tối thiểu 10), API Gateway APIs (tối thiểu 2) |
| Region | `us-east-1` khuyến nghị (AgentCore availability có thể khác) |

### IAM Permissions Cần Thiết

Role triển khai của bạn cần quyền cho:

- **AgentCore**: `bedrock-agentcore:*`
- **Lambda**: Tạo functions, quản lý IAM execution roles
- **API Gateway**: Tạo REST APIs, triển khai stages
- **Cognito**: Tạo user pools và identity pools
- **DynamoDB**: Tạo tables
- **S3**: Tạo buckets, put/get objects
- **SSM**: Tạo/đọc/cập nhật parameters
- **Secrets Manager**: Tạo và đọc secrets
- **ECR**: Push Docker images
- **CloudFormation**: Tạo/cập nhật/xóa stacks

## Công Cụ Local

| Công Cụ | Phiên Bản | Mục Đích |
|------|---------|---------|
| **Node.js** | 22.x | Lambda runtime, Nx build orchestration |
| **pnpm** | 9.x | Package manager (workspaces) |
| **Python** | 3.12+ | Agent code runtime |
| **uv** | latest | Python package manager |
| **Docker** | latest | Build agent container images |
| **AWS CLI** | latest | Cấu hình credentials, xác minh triển khai |
| **Hugo** | 0.134.3 | Xem trước workshop local (trang này) |

### Cài Đặt

```bash
# Node.js + pnpm
npm install -g pnpm@9

# Python + uv
pip install uv

# AWS CLI
# Xem: https://aws.amazon.com/cli/

# Xác minh
node --version  # v22.x
pnpm --version  # 9.x
python --version  # 3.12+
uv --version
aws --version
docker --version
```

## AWS Credentials

Cấu hình AWS CLI với profile có các quyền trên:

```bash
aws configure --profile mar-tech
```

Sau đó export profile trước khi triển khai:

```bash
export AWS_PROFILE=mar-tech
```

## Quyền Truy Cập Bedrock Model

Đảm bảo tài khoản AWS của bạn có quyền truy cập vào các Bedrock models bạn định dùng:

1. Mở **Amazon Bedrock** console
2. Vào **Model access** trong menu bên trái
3. Yêu cầu quyền truy cập ít nhất một model (ví dụ: `Claude Sonnet 4`)
4. Ghi lại model ID (ví dụ: `us.anthropic.claude-sonnet-4-20250514-v1:0`)

## Chi Phí AWS Ước Tính

| Dịch Vụ | Chi Phí Tháng Ước Tính | Ghi Chú |
|---------|----------------------|-------|
| AgentCore Runtime | ~$50-100 | 4 agent, sử dụng dev low-traffic |
| AgentCore Memory | ~$5-10 | Lưu trữ mỗi phiên |
| Lambda + API Gateway | ~$5-20 | Sử dụng API nhẹ |
| Bedrock model inference | ~$10-50 | Biến động theo token count |
| DynamoDB | ~$5 | PAY_PER_REQUEST, khối lượng thấp |
| S3 | ~$1 | Lưu trữ artifact |
| Cognito | Free tier | Dưới 50K MAU |
| CloudFront | Free tier | Dưới 1TB/tháng |
| **Tổng (ước tính)** | **~$76-186/tháng** | Sử dụng phát triển/demo |

## Dọn Dẹp

Để tránh phát sinh chi phí, xóa stack khi không sử dụng:

```bash
# Xóa CloudFormation stack
pnpm exec nx destroy @play-c463-z26-rzy-mar-tech/infra "stack-name/*"

# Xóa thủ công S3 buckets nếu CloudFormation thất bại
aws s3 rm s3://<sessions-bucket> --recursive
aws s3 rb s3://<sessions-bucket>
aws s3 rm s3://<sql-results-bucket> --recursive
aws s3 rb s3://<sql-results-bucket>
aws s3 rm s3://<access-logs-bucket> --recursive
aws s3 rb s3://<access-logs-bucket>

# Xóa CloudWatch log groups
aws logs delete-log-group --log-group-prefix "/aws/lambda/mar-tech-"
```
