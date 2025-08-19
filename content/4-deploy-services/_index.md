+++
title = "Deploy All Services with Serverless Framework"
weight = 4
+++

# Deploy All Services with Serverless Framework

## What you'll do
- Deploy all backend services using the Serverless Framework (Auth, Message, Real-time, Connect)
- Capture API Gateway endpoints and WebSocket URL
- Set post-deploy SSM parameters and (where needed) re‑deploy to refresh environment values
- Update the UI `.env` and run a quick end‑to‑end verification

## Learning outcomes
- Understand the deployment order and its rationale for dependencies
- Confidently deploy and re‑deploy services via a single CLI command
- Verify and troubleshoot common issues (CORS, SSM, Redis/VPC, Cognito auth)

---

### 1) Deployment order

Recommended sequence (minimizes missing resource references during first runs):

1. `authService` – creates `Users` table and auth endpoints
2. `messageService` – creates `Messages` table and HTTP endpoints
3. `realTimeService` – creates WebSocket API, `websocket-connections`, `conversation-members-table`, and `chat-message-topic`
4. `connectService` – creates `conversations` table and HTTP endpoints

{{% notice info %}}
The real-time service exposes CloudFormation Outputs including the WebSocket endpoint. After deploying it the first time, set `WEBSOCKET_DOMAIN` and `SNS_TOPIC_ARN` in SSM, then re‑deploy `realTimeService` to ensure Lambdas pick up the values as environment variables (if your code resolves from env at init time).
{{% /notice %}}

---

### 2) Deploy Auth Service

From the repository root `Serverless/`:

```bash
cd authService
serverless deploy
cd ..
```
![authService_Deploy](/images/authService_deploy.png)

Notes:
- This service requires `AWS_REGION` to be set in your shell during deploy (see Prerequisites).
- Capture the base HTTP API URL. Re‑print anytime with:


```bash
serverless info --verbose
```

---

### 3) Deploy Message Service

```bash
cd messageService
serverless deploy
cd ..
```

![messageService_Deploy](/images/messageService_deploy.png)

Notes:
- Creates the `Messages` table with GSI `conversationIdIndex`.
- Capture the base HTTP API URL (`/sendMessage`, `/getMessage`, ...).

---

### 4) Deploy Real-time Service (WebSocket)

```bash
cd realTimeService
serverless deploy
cd ..
```

Notes:
- Requires working VPC subnets and security group as configured earlier.
- Creates WebSocket API and tables `websocket-connections`, `conversation-members-table`, plus SNS topic `chat-message-topic`.
- Copy the WebSocket endpoint from stack outputs (looks like `wss://<api-id>.execute-api.ap-south-1.amazonaws.com/dev`).

---

### 5) Deploy Connect Service

```bash
cd connectService
serverless deploy
cd ..
```

Notes:
- Creates the `conversations` table with GSIs (`createdByIndex`, `createdAtIndex`).
- Capture the base HTTP API URL for conversation management endpoints.

---

### 6) Set post-deploy SSM parameters (WebSocket + SNS)

After the first successful `realTimeService` deployment:

1) Set `WEBSOCKET_DOMAIN` using the WebSocket endpoint you copied (exclude the stage segment)

```bash
# Example: endpoint is wss://abc123.execute-api.ap-south-1.amazonaws.com/dev
aws ssm put-parameter --name "/WEBSOCKET_DOMAIN" \
  --value "abc123.execute-api.ap-south-1.amazonaws.com" \
  --type String --overwrite
```

2) Set `SNS_TOPIC_ARN` to the ARN of `chat-message-topic` created by `realTimeService`:

```bash
aws ssm put-parameter --name "/SNS_TOPIC_ARN" \
  --value "arn:aws:sns:ap-south-1:<account-id>:chat-message-topic" \
  --type String --overwrite
```

3) Re‑deploy `realTimeService` so the latest SSM values are injected into Lambda environments (if your handlers read them as env vars):

```bash
cd realTimeService && serverless deploy && cd ..
```

{{% notice tip %}}
Alternatively, if your code fetches SSM parameters at runtime (not only at deploy time), a redeploy may not be strictly required. The provided configuration grants `ssm:GetParameter` permissions.
{{% /notice %}}

---

### 7) Verify deployed endpoints quickly

Replace `<base-url>` with your service HTTP API URL; include `Authorization: Bearer <jwt>` where required.

```bash
# Auth: signUp (public) -> confirmSignUp -> signIn
curl -X POST <auth-base-url>/signUp \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"P@ssw0rd!","fullName":"User"}'

# Message: sendMessage
curl -X POST <message-base-url>/sendMessage \
  -H "Authorization: Bearer <jwt>" -H "Content-Type: application/json" \
  -d '{"conversationId":"<id>","content":"Hello"}'

# Connect: createConversation
curl -X POST <connect-base-url>/createConversation \
  -H "Authorization: Bearer <jwt>" -H "Content-Type: application/json" \
  -d '{"name":"Demo","type":"group","memberIds":["<userId>"]}'
```

WebSocket quick test (using `wscat`):

```bash
wscat -c wss://<api-id>.execute-api.ap-south-1.amazonaws.com/dev
# Then send an authenticate message if your backend expects it
{"action":"authenticate","token":"<jwt>"}
```

---

### 8) Update the UI `.env` and run

From `telegrama-ui/README.md`, fill these variables using the deployed API Gateway URLs and WebSocket endpoint:

```env
VITE_AUTH_API_BASE_URL=https://<auth-api-id>.execute-api.ap-south-1.amazonaws.com
VITE_MESSAGE_API_BASE_URL=https://<message-api-id>.execute-api.ap-south-1.amazonaws.com
VITE_CONNECT_API_BASE_URL=https://<connect-api-id>.execute-api.ap-south-1.amazonaws.com
VITE_WEBSOCKET_URL=wss://<ws-api-id>.execute-api.ap-south-1.amazonaws.com/dev
```

Run locally:

```bash
cd telegrama-ui
npm run dev
```

{{% notice warning %}}
`VITE_CONNECT_API_BASE_URL` is required for user search and direct conversation features. Without it, those features will not work properly.
{{% /notice %}}

---

### 9) What was created automatically

- DynamoDB tables:
  - `Users` (Auth)
  - `Messages` with GSI `conversationIdIndex` (Message)
  - `websocket-connections` with GSI `userIdIndex` (Real-time)
  - `conversation-members-table` (Real-time)
  - `conversations` with GSIs `createdByIndex`, `createdAtIndex` (Connect)
- API Gateways:
  - HTTP APIs for Auth, Message, Connect
  - WebSocket API for Real-time with routes: `$connect`, `$disconnect`, `$default`, `message`, `typing`
- SNS topic: `chat-message-topic` (Real-time)

---

### 10) Troubleshooting

- Missing SSM parameter: Use `aws ssm get-parameter --name "/<PARAM>"` to verify. Set or fix with `--overwrite`.
- CORS errors: Ensure your UI origin is included (e.g., `http://localhost:5173`). CORS for common origins is preconfigured in each `serverless.yml`.
- 401/403 responses: Verify JWT and Cognito settings (`CLIENT_ID`, `COGNITO_USER_POOL_ID`).
- Redis/VPC connectivity: Confirm Lambda subnets and security group allow egress to ElastiCache on port 6379 and that routing/NAT is correct.
- WebSocket auth/management API: Double‑check `WEBSOCKET_DOMAIN` and `WEBSOCKET_STAGE` in SSM; re‑deploy `realTimeService` if these are read via env.

---

Once verified, proceed to Monitoring & Operations.
