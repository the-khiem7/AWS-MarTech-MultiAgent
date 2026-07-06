---
title: "Prerequisites & Setup"
date: 2026-07-06
weight: -1
chapter: false
pre: " <b> 0. </b> "
---

Before deploying the MarTech multi-agent platform, ensure your environment and AWS account are properly configured.

## AWS Account Requirements

| Resource | Minimum Required |
|----------|-----------------|
| AWS Account | Active account with billing enabled |
| IAM Permissions | Admin-level or the specific permissions listed below |
| Service Quotas | Bedrock model access, Lambda concurrency (at least 10), API Gateway APIs (at least 2) |
| Region | `us-east-1` recommended (AgentCore availability may vary) |

### Required IAM Permissions

Your deployment role needs permissions for:

- **AgentCore**: `bedrock-agentcore:*`
- **Lambda**: Create functions, manage IAM execution roles
- **API Gateway**: Create REST APIs, deploy stages
- **Cognito**: Create user pools and identity pools
- **DynamoDB**: Create tables
- **S3**: Create buckets, put/get objects
- **SSM**: Create/read/update parameters
- **Secrets Manager**: Create and read secrets
- **ECR**: Push Docker images
- **CloudFormation**: Create/update/delete stacks

{{% notice tip %}}
The easiest way to get started is to use an IAM role with the **AdministratorAccess** managed policy, then scope down after deployment.
{{% /notice %}}

## Local Tooling

| Tool | Version | Purpose |
|------|---------|---------|
| **Node.js** | 22.x | Lambda runtime, Nx build orchestration |
| **pnpm** | 9.x | Package manager (workspaces) |
| **Python** | 3.12+ | Agent code runtime |
| **uv** | latest | Python package manager |
| **Docker** | latest | Build agent container images |
| **AWS CLI** | latest | Configure credentials, verify deployment |
| **Hugo** | 0.134.3 | Local workshop preview (this site) |

### Installation

```bash
# Node.js + pnpm
npm install -g pnpm@9

# Python + uv
pip install uv

# AWS CLI
# See: https://aws.amazon.com/cli/

# Verify
node --version  # v22.x
pnpm --version  # 9.x
python --version  # 3.12+
uv --version
aws --version
docker --version
```

## AWS Credentials

Configure your AWS CLI with a profile that has the permissions above:

```bash
aws configure --profile mar-tech
```

Then export the profile before deploying:

```bash
export AWS_PROFILE=mar-tech
```

## Bedrock Model Access

Ensure your AWS account has access to the Bedrock models you plan to use:

1. Open the **Amazon Bedrock** console
2. Navigate to **Model access** in the left menu
3. Request access to at least one model (e.g., `Claude Sonnet 4` or similar)
4. Note the model ID (e.g., `us.anthropic.claude-sonnet-4-20250514-v1:0`)

## Estimated AWS Costs

| Service | Estimated Monthly Cost | Notes |
|---------|----------------------|-------|
| AgentCore Runtime | ~$50-100 | 4 agents, low-traffic dev usage |
| AgentCore Memory | ~$5-10 | Per-session storage |
| Lambda + API Gateway | ~$5-20 | Light API usage |
| Bedrock model inference | ~$10-50 | Variable by token count |
| DynamoDB | ~$5 | PAY_PER_REQUEST, low volume |
| S3 | ~$1 | Artifact storage |
| Cognito | Free tier | Under 50K MAU |
| CloudFront | Free tier | Under 1TB/month |
| **Total (estimated)** | **~$76-186/month** | Development/demo usage |

{{% notice warning %}}
Costs scale with usage. Production deployments with high message volumes or large Bedrock model invocations will cost more. Use the [AWS Pricing Calculator](https://calculator.aws/) for precise estimates.
{{% /notice %}}

## Cleanup

To avoid ongoing charges, tear down the stack when not in use:

```bash
# Delete the CloudFormation stack (this removes all provisioned resources)
pnpm exec nx destroy @play-c463-z26-rzy-mar-tech/infra "stack-name/*"

# Manually empty and delete S3 buckets if CloudFormation fails
aws s3 rm s3://<sessions-bucket> --recursive
aws s3 rb s3://<sessions-bucket>
aws s3 rm s3://<sql-results-bucket> --recursive
aws s3 rb s3://<sql-results-bucket>
aws s3 rm s3://<access-logs-bucket> --recursive
aws s3 rb s3://<access-logs-bucket>

# Delete CloudWatch log groups
aws logs delete-log-group --log-group-prefix "/aws/lambda/mar-tech-"
```

{{% notice info %}}
Bedrock model access requests do not incur charges by themselves. You are only charged for inference calls made by the agents during testing.
{{% /notice %}}
