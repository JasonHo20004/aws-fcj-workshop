+++
title = "Cleanup resources"
weight = 6
+++

## What you'll do
- Remove all Serverless stacks (Auth, Message, Real-time, Connect)
- Clean up SSM parameters used by the services
- Optionally delete CloudWatch log groups
- Tear down ElastiCache Redis (created outside stacks)

## Learning outcomes
- Be able to fully deprovision the workshop resources and stop incurring costs

---

### 1) Remove the Serverless services

From the repository root `Serverless/`, run for each service:

```bash
cd realTimeService && serverless remove && cd ..
cd messageService && serverless remove && cd ..
cd connectService && serverless remove && cd ..
cd authService && serverless remove && cd ..
```

This deletes:
- DynamoDB tables: `websocket-connections`, `conversation-members-table`, `Messages`, `conversations`, `Users`
- API Gateways (HTTP/WebSocket) and related routes (`$connect`, `$disconnect`, `$default`, `message`, `typing`)
- SNS topic `chat-message-topic` (owned by `realTimeService` stack)

{{% notice warning %}}
Occasionally, CloudWatch log groups remain after stack removal. See step 3 to clean them up.
{{% /notice %}}

---

### 2) Delete SSM Parameter Store entries

Parameters used across services:

```bash
/CLIENT_ID
/COGNITO_USER_POOL_ID
/COGNITO_JWT_SECRET           # optional
/USERS_TABLE_NAME             # Users
/MESSAGE_TABLE_NAME           # Messages
/WEBSOCKET_TABLE_NAME         # websocket-connections
/CONVERSATION_MEMBERS_TABLE   # conversation-members-table
/CONVERSATIONS_TABLE_NAME     # conversations
/WEBSOCKET_STAGE              # e.g., dev
/WEBSOCKET_DOMAIN             # from deployed WebSocket endpoint (domain only)
/SNS_TOPIC_ARN                # created by realTimeService
/ENABLE_METRICS               # optional flag
/MAX_CONNECTIONS_PER_USER     # optional
/CONNECTION_CACHE_TTL         # optional
/MESSAGE_ACK_TIMEOUT          # optional
/MESSAGE_TTL_DAYS             # optional
```

Delete them (adjust list to what you actually set):

```bash
for p in \
  "/CLIENT_ID" \
  "/COGNITO_USER_POOL_ID" \
  "/COGNITO_JWT_SECRET" \
  "/USERS_TABLE_NAME" \
  "/MESSAGE_TABLE_NAME" \
  "/WEBSOCKET_TABLE_NAME" \
  "/CONVERSATION_MEMBERS_TABLE" \
  "/CONVERSATIONS_TABLE_NAME" \
  "/WEBSOCKET_STAGE" \
  "/WEBSOCKET_DOMAIN" \
  "/SNS_TOPIC_ARN" \
  "/ENABLE_METRICS" \
  "/MAX_CONNECTIONS_PER_USER" \
  "/CONNECTION_CACHE_TTL" \
  "/MESSAGE_ACK_TIMEOUT" \
  "/MESSAGE_TTL_DAYS"; do
  aws ssm delete-parameter --name "$p" || true
done
```

---

### 3) Optionally remove CloudWatch log groups

Serverless may leave function log groups behind. You can delete by prefix via CLI (double‑check names in the console first):

```bash
# Example for realTimeService (adjust stage/region/account)
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/realTimeService-dev-" \
  --query "logGroups[].logGroupName" --output text | \
  ForEach-Object { aws logs delete-log-group --log-group-name $_ }
```

{{% notice tip %}}
On macOS/Linux, replace the last line with `xargs -n1 aws logs delete-log-group --log-group-name`.
{{% /notice %}}

If used, also remove the Redis log group:

```bash
aws logs delete-log-group --log-group-name "/aws/elasticache/realtime-redis" || true
```

---

### 4) Tear down ElastiCache Redis (manual)

ElastiCache resources were created outside of CloudFormation. In the AWS Console:
- ElastiCache → Redis → find cluster `realtime-redis-cluster` and delete it
- If you created a dedicated subnet group (e.g., `realtime-redis-subnet-group`), delete it
- Ensure the security group you used is not shared with other workloads before removing

---

### 5) UI cleanup (if deployed)

If you deployed the UI to S3/CloudFront:
- Empty and delete the S3 bucket
- Disable and delete the CloudFront distribution

---

### 6) Final cost check

Use Cost Explorer to confirm spend drops after teardown. Leave the account running for a day to ensure hourly resources are fully stopped.
