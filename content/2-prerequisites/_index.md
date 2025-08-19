+++
title = "Prerequisites & Environment Setup"
weight = 2
+++

## What you'll do
- Install tools: Node.js, npm, AWS CLI, Serverless Framework, Git
- Configure AWS credentials and set region `ap-south-1`
- Create an IAM user (workshop-friendly permissions)
- Clone the repo and install dependencies for each service and the UI
- Configure `.env` for the UI and SSM Parameter Store for the backend
- Install helpful tools: Postman, Redis CLI (optional)
- Run quick pre-deploy checks

## Learning outcomes
- Have a complete environment ready for deployment
- Understand the configuration/SSM parameters used by the system
- Be ready to deploy all services with the Serverless Framework

---

### 1) Install required tools

- **Node.js (22 is recommended) & npm**: download from `https://nodejs.org` and verify
```bash
node --version
npm --version
```
![Node.js version](/images/node-npm_verify.png)

- **AWS CLI**: download from `https://aws.amazon.com/cli/` and verify
```bash
aws --version
```
![AWS CLI version](/images/aws-cli_verify.png)

- **Serverless Framework (CLI)**
```bash
npm install -g serverless
serverless --version
```
![Serverless Framework](/images/social-card-serverless-framework-1a.png)

Why we use the Serverless Framework in this workshop:
- It provides Infrastructure-as-Code for AWS Lambda, API Gateway (HTTP/WebSocket), DynamoDB, SNS, and SSM in one place (`serverless.yml`).
- Consistent, repeatable deployments with a single command per service: `serverless deploy` and easy teardown with `serverless remove`.
- Built-in support for stages, variables, CORS, and environment/SSM parameter resolution.
- Clear IAM scoping via `iamRoleStatements` and automatic packaging/log management.
- Great fit for a multi-service, event-driven architecture like this chat system.
- Learn more at `https://www.serverless.com/`.

- **Git**: `https://git-scm.com/`

{{% notice tip %}}
We recommend installing **Postman** (https://www.postman.com/downloads/) for testing HTTP APIs and **Redis CLI** (https://redis.io/download) for testing Redis connectivity.

{{% /notice %}}

---

### 2) Configure AWS credentials and region

Run the command and enter your Access Key/Secret Key (use the workshop account):
```bash
aws configure
# Region: ap-south-1
```

- Windows PowerShell (specific for `authService` since its `serverless.yml` reads this env var):
```powershell
$env:AWS_REGION = "ap-south-1"
```
- macOS/Linux equivalent:
```bash
export AWS_REGION=ap-south-1
```

![AWS Configure](/images/aws_config.png)

I have already configured AWS credentials for this workshop using `aws configure`. The configuration includes:

- AWS Access Key ID and Secret Access Key from your account
- Default region set to `ap-south-1` 

> Note: Other services already fix the region to `ap-south-1` in their `serverless.yml`.

---

### 3) IAM permissions for deployment

For workshop simplicity, you can use broad permissions: **AdministratorAccess**. For a minimal set, you need permissions for: CloudFormation, S3 (deployment artifacts), IAM (Lambda roles), Lambda, API Gateway (HTTP/WebSocket), DynamoDB, SSM Parameter Store, SNS, and EC2 (describe network for VPC).

---

### 4) Clone the repository and install dependencies

```bash
# Clone the repo (if you haven't)
git clone https://github.com/JasonHo20004/FCJ-workshop.git
cd Serverless

```

```bash
# Install for each backend service
cd authService && npm install && cd ..
cd messageService && npm install && cd ..
cd realTimeService && npm install && cd ..
cd connectService && npm install && cd ..
```



---

### 5) Clone and configure the UI (separate repository)

Clone the UI repository and install dependencies:
```bash
git clone <telegrama-ui-repo-url>
cd telegrama-ui
npm install
```

Create a `.env` file and add the following variables:

```bash
VITE_AUTH_API_BASE_URL=https://your-api-gateway-url/auth
VITE_MESSAGE_API_BASE_URL=https://your-api-gateway-url/messages
VITE_CONNECT_API_BASE_URL=https://your-api-gateway-url/connect
VITE_WEBSOCKET_URL=wss://your-websocket-api-url/prod
```
{{% notice tip %}}
You can set these environment variables later after deploying all backend services. The actual values will be available from the API Gateway endpoints and WebSocket URL.

{{% /notice %}}

{{% notice warning %}}
`VITE_CONNECT_API_BASE_URL` is required for user search and direct conversation features. Without this environment variable, these features will not work properly.
{{% /notice %}}

Run locally:
```bash
npm run dev  # serves on http://localhost:5173
```

Build for production and deploy the `dist/` folder to static hosting (e.g., S3 + CloudFront):
```bash
npm run build
```

---

### 6) Set up SSM Parameter Store (backend)

Services use SSM parameters for configuration. Set the following parameters. Use `--overwrite` when updating values.

Required BEFORE deploying backend services:
```bash
# Cognito
aws ssm put-parameter --name "/CLIENT_ID" --value "your-cognito-app-client-id" --type String --overwrite
aws ssm put-parameter --name "/COGNITO_USER_POOL_ID" --value "your-user-pool-id" --type String --overwrite
# Optional if you use an internal JWT verification secret
aws ssm put-parameter --name "/COGNITO_JWT_SECRET" --value "your-jwt-secret" --type String --overwrite

# DynamoDB table names
aws ssm put-parameter --name "/USERS_TABLE_NAME" --value "Users" --type String --overwrite
aws ssm put-parameter --name "/MESSAGE_TABLE_NAME" --value "Messages" --type String --overwrite
aws ssm put-parameter --name "/WEBSOCKET_TABLE_NAME" --value "websocket-connections" --type String --overwrite
aws ssm put-parameter --name "/CONVERSATION_MEMBERS_TABLE" --value "conversation-members-table" --type String --overwrite
aws ssm put-parameter --name "/CONVERSATIONS_TABLE_NAME" --value "conversations" --type String --overwrite

# WebSocket stage (used by realTimeService)
aws ssm put-parameter --name "/WEBSOCKET_STAGE" --value "dev" --type String --overwrite

# Redis (ElastiCache)
aws ssm put-parameter --name "/REDIS_HOST" --value "your-redis-endpoint" --type String --overwrite
aws ssm put-parameter --name "/REDIS_PORT" --value "6379" --type String --overwrite
```

Set AFTER deployment (values are only known once resources exist):
```bash
# WebSocket domain from API Gateway WebSocket endpoint (exclude stage)
aws ssm put-parameter --name "/WEBSOCKET_DOMAIN" --value "your-websocket-domain" --type String --overwrite

# SNS Topic ARN created by realTimeService (`chat-message-topic`)
aws ssm put-parameter --name "/SNS_TOPIC_ARN" --value "arn:aws:sns:ap-south-1:<account-id>:chat-message-topic" --type String --overwrite
```

> Important:
> - `authService` reads the `AWS_REGION` env var during deployment.
> - `realTimeService` uses `WEBSOCKET_DOMAIN` and `WEBSOCKET_STAGE` with API Gateway Management API; set `WEBSOCKET_DOMAIN` after the WebSocket API is deployed.
> - Set `SNS_TOPIC_ARN` after the SNS topic is created during `realTimeService` deployment.

Quick parameter checks:
```bash
aws ssm get-parameter --name "/CLIENT_ID"
aws ssm get-parameter --name "/WEBSOCKET_TABLE_NAME"
```

---

### 7) CORS and allowed UI origins

Services already enable CORS for:
- `https://<my-domain>.cloudfront.net`
- `http://localhost:5173`
- `http://localhost:3000`


{{% notice warning %}}
`VITE_CONNECT_API_BASE_URL` is required for user search and direct conversation features. Without this environment variable, these features will not work properly.
{{% /notice %}}


---

### 8) Pre-deploy quick checks

```bash
# Verify the AWS identity
aws sts get-caller-identity

# Tool versions
serverless --version
node --version && npm --version && aws --version

# Check a few key SSM parameters
aws ssm get-parameter --name "/MESSAGE_TABLE_NAME"
aws ssm get-parameter --name "/WEBSOCKET_STAGE"
```

---

Notes:
- Ensure the UI origin (e.g., `http://localhost:5173` or your CloudFront domain) is allowed by backend CORS (already configured for common dev/prod origins).
- Use `wss://` for `VITE_WEBSOCKET_URL` and verify it matches the deployed WebSocket stage.
- If requests fail, re-check the `.env` values and your API Gateway URLs.

Once done, you're ready to move on to Deploy Infrastructure & Services.
