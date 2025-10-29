# Day 1: ALB + Service Connect Implementation

## Step 0: Create IAM Roles (15 minutes) - **REQUIRED FIRST**

**⚠️ IMPORTANT:** Create IAM roles before task definitions!

### **Quick Commands for API Gateway Staging:**
```bash
# Create execution role
aws iam create-role \
  --role-name ic-apigateway-staging-execution-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ecs-tasks.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }' \
  --region ap-southeast-1

# Attach execution policies
aws iam attach-role-policy \
  --role-name ic-apigateway-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-apigateway-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

# Create task role
aws iam create-role \
  --role-name ic-apigateway-staging-task-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ecs-tasks.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }' \
  --region ap-southeast-1

# Create and attach task policy
aws iam create-policy \
  --policy-name ic-apigateway-staging-task-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject",
          "secretsmanager:GetSecretValue"
        ],
        "Resource": "*"
      }
    ]
  }' \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-apigateway-staging-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-apigateway-staging-task-policy \
  --region ap-southeast-1
```

**For all services:** See complete guide: [IAM Roles Creation](../08-iam-roles-setup/iam-roles-creation.md)

## Step 1: Create ECS Cluster (5 minutes)

```bash
# Create ECS cluster for IC microservices
aws ecs create-cluster \
  --cluster-name ic-microservices-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --region ap-southeast-1

# Verify cluster creation
aws ecs describe-clusters \
  --clusters ic-microservices-cluster \
  --region ap-southeast-1
```

## Step 2: Create Service Discovery Namespace (10 minutes)

```bash
# Create private DNS namespace for Service Connect
aws servicediscovery create-private-dns-namespace \
  --name ic-microservices \
  --vpc vpc-REPLACE_WITH_YOUR_VPC_ID \
  --region ap-southeast-1

# Save the namespace ID from output - you'll need it later
# Example output: "NamespaceId": "ns-abcd1234efgh5678"
```

## Step 2: Create Application Load Balancer (15 minutes)

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name ic-microservices-alb \
  --subnets subnet-REPLACE_PUBLIC_SUBNET_1 subnet-REPLACE_PUBLIC_SUBNET_2 \
  --security-groups sg-REPLACE_WITH_ALB_SECURITY_GROUP \
  --scheme internet-facing \
  --type application \
  --region ap-southeast-1

# Create target group for API Gateway
aws elbv2 create-target-group \
  --name api-gateway-tg \
  --protocol HTTP \
  --port 8000 \
  --vpc-id vpc-REPLACE_WITH_YOUR_VPC_ID \
  --target-type ip \
  --health-check-protocol TCP \
  --health-check-port 8000 \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --region ap-southeast-1

# Create HTTPS listener with SSL certificate
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:loadbalancer/app/ic-microservices-alb/REPLACE_WITH_ALB_ID \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:ap-southeast-1:795189341938:certificate/7cd7d7d8-f9ad-40b1-a0ca-e56472345fa4 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/api-gateway-tg/REPLACE_WITH_TG_ID \
  --region ap-southeast-1
```

## Step 3: Update API Gateway Task Definition (20 minutes)

**⚠️ IMPORTANT:** Client's API Gateway reads microservice URLs from **database**, not environment variables.
See: [Database Updates Guide](../09-database-updates/service-connect-database-migration.md)

Create file: `ic-apigateway-staging-task-definition.json`

```json
{
  "family": "ic-apigateway-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-task-role",
  "containerDefinitions": [
    {
      "name": "ic-api-gateway-container",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/ic-api-gateway-image:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "ic-api-gateway-port",
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        {
          "name": "APP_NAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:APP_NAME::"
        },
        {
          "name": "APP_ENV",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:APP_ENV::"
        },
        {
          "name": "APP_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:APP_KEY::"
        },
        {
          "name": "APP_DEBUG",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:APP_DEBUG::"
        },
        {
          "name": "APP_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:APP_URL::"
        },
        {
          "name": "APP_TIMEZONE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:APP_TIMEZONE::"
        },
        {
          "name": "LOG_CHANNEL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:LOG_CHANNEL::"
        },
        {
          "name": "LOG_SLACK_WEBHOOK_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:LOG_SLACK_WEBHOOK_URL::"
        },
        {
          "name": "DB_CONNECTION",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_CONNECTION::"
        },
        {
          "name": "DB_HOST",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_HOST::"
        },
        {
          "name": "DB_PORT",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_PORT::"
        },
        {
          "name": "DB_DATABASE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_DATABASE::"
        },
        {
          "name": "DB_USERNAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_USERNAME::"
        },
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_PASSWORD::"
        },
        {
          "name": "DB_TIMEZONE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:DB_TIMEZONE::"
        },
        {
          "name": "PROFILE_PICS_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:PROFILE_PICS_URL::"
        },
        {
          "name": "CACHE_DRIVER",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:CACHE_DRIVER::"
        },
        {
          "name": "QUEUE_CONNECTION",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:QUEUE_CONNECTION::"
        },
        {
          "name": "ACCEPTED_SECRETS",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:ACCEPTED_SECRETS::"
        },
        {
          "name": "FILES_MICROSERVICE_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:FILES_MICROSERVICE_URL::"
        },
        {
          "name": "FILES_MICROSERVICE_SECRET",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-api-gateway-staging-secrets-svwZog:FILES_MICROSERVICE_SECRET::"
        }
      ],
      "environment": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-api-gateway-staging-logs",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
```

**Note:** 
- No Service Connect URLs in environment variables - API Gateway reads from database
- All your existing secrets are preserved
- Uses your naming convention: `ic-api-gateway-port`

Register the task definition:
```bash
aws ecs register-task-definition \
  --cli-input-json file://ic-apigateway-staging-task-definition.json \
  --region ap-southeast-1
```

## Step 5: Create API Gateway ECS Service with Service Connect (20 minutes)

```bash
# Create API Gateway ECS service with Service Connect and ALB
aws ecs create-service \
  --cluster ic-microservices-cluster \
  --service-name ic-apigateway-staging \
  --task-definition ic-apigateway-staging-task \
  --desired-count 1 \
  --capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-096a5e3c10eef5f5c", "subnet-0e9a0ec15dc80197d"],
      "securityGroups": ["sg-0bcd67a1053a4e84a"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "REPLACE_WITH_NAMESPACE_ID_FROM_STEP2",
    "services": [
      {
        "portName": "ic-api-gateway-port",
        "discoveryName": "api-gateway",
        "clientAliases": [
          {
            "port": 8000,
            "dnsName": "api-gateway.local"
          }
        ]
      }
    ]
  }' \
  --load-balancers '[
    {
      "targetGroupArn": "arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/ic-apigateway-staging-tg/92e300ce623f18bf",
      "containerName": "ic-api-gateway-container",
      "containerPort": 8000
    }
  ]' \
  --region ap-southeast-1

# Verify service creation
aws ecs describe-services \
  --cluster ic-microservices-cluster \
  --services ic-apigateway-staging \
  --region ap-southeast-1
```

## Step 6: Test ALB Connection (10 minutes)

```bash
# Get ALB DNS name
aws elbv2 describe-load-balancers \
  --names ic-apigateway-staging-lb \
  --query 'LoadBalancers[0].DNSName' \
  --output text \
  --region ap-southeast-1

# Test HTTPS connection (replace with actual ALB DNS name)
curl -k https://YOUR_ALB_DNS_NAME_FROM_ABOVE_COMMAND

# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/ic-apigateway-staging-tg/92e300ce623f18bf \
  --region ap-southeast-1

# Expected: Target should show "healthy" status
```

## Day 1 Completion Checklist
- ✅ IAM roles created for API Gateway staging
- ✅ ECS cluster `ic-microservices-cluster` created
- ✅ Service Discovery namespace created
- ✅ Application Load Balancer created with SSL certificate
- ✅ Target group created with TCP health checks
- ✅ API Gateway task definition registered
- ✅ API Gateway ECS service created with Service Connect and ALB
- ✅ HTTPS connection working through ALB

**Next:** Day 2 - Create remaining services (AUTH, CORE, FILES) + CI/CD setup
