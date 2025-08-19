+++
title = "Monitoring & Operations"
weight = 5
+++

## What you'll do
- Use CloudWatch Logs and the Serverless CLI to inspect/trace requests end-to-end
- Enable and view custom metrics for the real-time pipeline
- Create basic CloudWatch Alarms for errors and throttling
- Validate Redis connectivity and WebSocket health

## Learning outcomes
- Know where to find logs and metrics for each service
- Be able to quickly tail logs and correlate requests
- Understand a minimal alerting setup for production readiness

---

### 1) Where telemetry lives

- **Lambda logs (default)**: CloudWatch Logs → Log groups → /aws/lambda/<function-name>
  - authService: signUp, confirmSignUp, resendConfirmationCode, signIn, signOut
  - messageService: fetchMessage, sendMessage, deleteMessage, getMessage, updateMessage, editMessage
  - realTimeService (WebSocket): connect, disconnect, $default, message, typing, snsFanoutHandler, redisSubscriber
  - connectService: createConversation, getConversations, getConversation, addMember, removeMember, getMembers, updateConversation, deleteConversation, searchUsers, createDirectConversation

- **Redis/ElastiCache log group** (predefined): /aws/elasticache/realtime-redis

- **Built-in metrics** (CloudWatch → Metrics):
  - Lambda: Invocations, Errors, Duration, Throttles
  - DynamoDB tables: ReadThrottleEvents, WriteThrottleEvents
  - API Gateway (HTTP/WebSocket): 4XXError, 5XXError, ConnectCount, MessageCount (if enabled in your account)

  ![Log_Groups](/images/log_groups.png)

{{% notice info %}}
API Gateway access logs are not explicitly configured in these services; rely on Lambda logs for request/response traces.
{{% /notice %}}

---

### 2) Tail logs quickly (Serverless CLI)

Run from each service directory:

```bash
# Example: tail real-time message route logs
cd realTimeService
serverless logs --function message --tail

# WebSocket $default route
serverless logs --function default --tail

# SNS fan-out handler
serverless logs --function snsFanoutHandler --tail

# Message service sendMessage
cd ../messageService
serverless logs --function sendMessage --tail
```

Common things to verify in logs:
- JWT auth success/failure and user identifiers
- DynamoDB read/write outcomes and consumed capacity
- WebSocket `postToConnection` responses and disconnects
- Redis connection success (host/port from SSM) and timeouts

---

### 3) Enable custom metrics (realTimeService)

`realTimeService` is permissioned to publish custom metrics. Toggle metrics with SSM:

```bash
# Turn on custom metrics for real-time handlers
aws ssm put-parameter --name "/ENABLE_METRICS" --value "true" --type String --overwrite

# Re-deploy so functions read updated env (if resolved at init time)
cd realTimeService && serverless deploy && cd ..
```

Look for a custom namespace (e.g., `RealTimeChat`) and metrics like `MessagesPublished`, `FanoutFailures`, `ActiveConnections`. Use the Metrics explorer to graph alongside Lambda `Errors` and `Duration`.

{{% notice tip %}}
If your code fetches SSM parameters at runtime for each invocation, redeploy may not be required. These Lambdas already have `ssm:GetParameter` permissions.
{{% /notice %}}

---

### 4) Create basic CloudWatch alarms

Create a small alerting baseline. First, create an SNS topic for notifications and subscribe your email:

```bash
# Create an alarm topic
aws sns create-topic --name workshop-alarms

# Subscribe your email
aws sns subscribe --topic-arn arn:aws:sns:ap-south-1:<account-id>:workshop-alarms \
  --protocol email --notification-endpoint you@example.com
```

- **Lambda Errors (realTimeService/message route)**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rt-message-errors \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum --period 300 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --dimensions Name=FunctionName,Value=realTimeService-dev-message \
  --alarm-actions arn:aws:sns:ap-south-1:<account-id>:workshop-alarms
```

- **DynamoDB throttling (Messages table)**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ddb-messages-throttles \
  --metric-name WriteThrottleEvents \
  --namespace AWS/DynamoDB \
  --statistic Sum --period 300 --threshold 1 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=TableName,Value=Messages \
  --alarm-actions arn:aws:sns:ap-south-1:<account-id>:workshop-alarms
```

{{% notice tip %}}
You can also add alarms for API Gateway `5XXError` and Lambda `Throttles`. Use the console to find exact metric dimensions (API ID, Stage) in your account.
{{% /notice %}}

---

### 5) Quick health checks and SSM verification

```bash
# Verify required SSM parameters
aws ssm get-parameter --name "/WEBSOCKET_DOMAIN"
aws ssm get-parameter --name "/WEBSOCKET_STAGE"
aws ssm get-parameter --name "/SNS_TOPIC_ARN"

# Check Redis configuration
aws ssm get-parameter --name "/REDIS_HOST"
aws ssm get-parameter --name "/REDIS_PORT"
```

Troubleshooting hints:
- 401/403 responses: re-check JWT (Cognito `CLIENT_ID`, `COGNITO_USER_POOL_ID`) and token audience
- WebSocket send failures: confirm `WEBSOCKET_DOMAIN` and stage are correct; verify connections exist in `websocket-connections` (GSI `userIdIndex`)
- Fan-out lag: check `snsFanoutHandler` logs and SNS delivery status; verify `SNS_TOPIC_ARN` value
- DynamoDB hot partitions: watch `ConsumedRead/Write` and `ThrottledRequests` on `Messages` and `conversations`

---

### 6) Optional: simple dashboard

Create a quick dashboard combining key metrics (Lambda Errors/Duration, DynamoDB throttles, WebSocket message counts). This can be done via the CloudWatch console → Dashboards in a few clicks using the metrics mentioned above.

Once you are comfortable with monitoring, proceed to Cleanup resources.
