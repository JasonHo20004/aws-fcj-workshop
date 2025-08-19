+++
title = "Deploy Infrastructure & Services"
weight = 3
+++

# Deploy Infrastructure & Services

## What you'll do
- Prepare networking (VPC subnets and security group) for Lambda and Redis
- Provision or connect to an existing ElastiCache Redis cluster
- Configure required SSM parameters for infrastructure
- Understand which resources will be created automatically when deploying services in the next step

## Learning outcomes
- Understand how Redis, VPC networking, and SSM configuration fit together
- Be able to provide Redis endpoint and stage-level parameters
- Know which DynamoDB tables, SNS topics, and WebSocket APIs will be auto-created

---

### 1) Networking & VPC prerequisites

The real-time WebSocket service runs inside a VPC and needs subnets and a security group. The IDs are configured in `realTimeService/serverless.yml`.

```yaml
provider:
  vpc:
    securityGroupIds:
      - sg-078a8807b2abd8a28
    subnetIds:
      - subnet-0ccd1edb1f1c360b4
      - subnet-0b7d121bdc8c1acd7
      - subnet-05affc69b84163099
```

{{% notice tip %}}
Since these workshop resources are temporary, replace the security group and subnet IDs with ones from your own AWS account, ensuring the security group permits outbound traffic to Redis (port 6379) and the private subnets have proper network routing to ElastiCache through NAT Gateway or VPC endpoints.
{{% /notice %}}

---

### 2) Set up ElastiCache Redis

Follow the console steps shown in the screenshots to create a Redis cluster inside your VPC.

1) In AWS Console, go to ElastiCache > Valkey caches > Create cache

![Redis-Initial-Step](/images/elasti-cache-initial.png)


2) Choose the following settings as shown in the image below for cost optimization:
   - Subnet group: Select your VPC's private subnets
   - Security group: Use the same security group will configure for Lambda

![Redis-Setup-Steps](/images/redis_creation.jpeg)

2) Wait until the cluster is available, open its details, and copy the Primary endpoint

- Primary endpoint will look like: `master.<cluster>.aps1.cache.amazonaws.com`

![Setup Redis Complete](/images/my-redis_setup_complete.png)

{{% notice tip %}}
Take note of the following information in the above image for later setup.
{{% /notice %}}

Set the Redis endpoint and port in SSM:

```bash
aws ssm put-parameter --name "/REDIS_HOST" --value "your-redis-endpoint" --type String --overwrite
aws ssm put-parameter --name "/REDIS_PORT" --value "6379" --type String --overwrite
```

{{% notice info %}}
If your cluster requires AUTH, also set `/REDIS_PASSWORD`.
{{% /notice %}}

---

### 3) Configure infrastructure SSM parameters

Set or confirm these parameters. Some were introduced in the previous step; list here for convenience.

```bash
# WebSocket stage used by API Gateway Management API
aws ssm put-parameter --name "/WEBSOCKET_STAGE" --value "dev" --type String --overwrite

```

Parameters to set AFTER deployment (values only known once resources exist):

```bash
# SNS Topic ARN created by the real-time service (chat-message-topic)
aws ssm put-parameter --name "/SNS_TOPIC_ARN" --value "arn:aws:sns:ap-south-1:<account-id>:chat-message-topic" --type String --overwrite
```

---

### 4) IAM roles & policies (already defined)

IAM permissions are scoped per service via `serverless.yml`. You don't need to create roles manually.

You can review and customize these in each service's `serverless.yml` under `iamRoleStatements`.

![IAMRoleStatement](/images/iamrolestatement_example.png)

---

### 5) What will be auto-created when deploying services (next step)

You do not need to create DynamoDB tables or the SNS topic manually. They are defined in the services' `serverless.yml` and will be created by CloudFormation when you deploy.

- DynamoDB tables:
  - Users table (AuthService)
  - Conversations table and `conversation-members-table` (ConnectService / RealTimeService)
  - Messages table with GSI `conversationIdIndex` (MessageService)
  - WebSocket connections table `websocket-connections` with GSI `userIdIndex` (RealTimeService)
- SNS topic:
  - `chat-message-topic` (RealTimeService)
- WebSocket API Gateway with routes `$connect`, `$disconnect`, `$default`, `message`, `typing` (RealTimeService)

{{% notice warning %}}
Ensure SSM parameters (Redis, stage, table names) are set before deploying services. Missing parameters will cause deployment or runtime failures.
{{% /notice %}}

---

### 6) Verify readiness

Quick checks before moving on:

```bash
# Confirm required SSM parameters exist
aws ssm get-parameter --name "/REDIS_HOST" --with-decryption
aws ssm get-parameter --name "/REDIS_PORT"
aws ssm get-parameter --name "/WEBSOCKET_STAGE"

# (Optional) If you created ElastiCache, verify its status in the console or via CLI
# aws elasticache describe-cache-clusters --show-cache-node-info | rg <your-cluster-id>
```

Once everything looks good, continue to the next page to deploy all services with the Serverless Framework.
