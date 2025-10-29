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

## Step 1: Create Service Discovery Namespace (10 minutes)

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

Create file: `ic-apigateway-staging-task-definition.json`

```json
{
  "family": "ic-apigateway-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-task-role",
  "containerDefinitions": [
    {
      "name": "api-gateway",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/api-gateway:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "api-port",
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        {
          "name": "DB_HOST",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:host::"
        },
        {
          "name": "DB_USERNAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:username::"
        },
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:password::"
        }
      ],
      "environment": [
        {
          "name": "DB_SCHEMA",
          "value": "staging_employees"
        },
        {
          "name": "AUTH_SERVICE_URL",
          "value": "http://auth-service.local:8001"
        },
        {
          "name": "CORE_SERVICE_URL",
          "value": "http://core-service.local:8002"
        },
        {
          "name": "FILES_SERVICE_URL",
          "value": "http://files-service.local:8003"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-apigateway-staging",
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

Register the task definition:
```bash
aws ecs register-task-definition \
  --cli-input-json file://ic-apigateway-staging-task-definition.json \
  --region ap-southeast-1
```

## Step 4: Update API Gateway ECS Service with Service Connect (15 minutes)

```bash
# Update existing API Gateway service to use Service Connect and ALB
aws ecs update-service \
  --cluster REPLACE_WITH_YOUR_CLUSTER_NAME \
  --service REPLACE_WITH_API_GATEWAY_SERVICE_NAME \
  --task-definition ic-apigateway-staging-task \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "REPLACE_WITH_NAMESPACE_ID_FROM_STEP1",
    "services": [
      {
        "portName": "api-port",
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
      "targetGroupArn": "arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/api-gateway-tg/REPLACE_WITH_TG_ID",
      "containerName": "api-gateway",
      "containerPort": 8000
    }
  ]' \
  --region ap-southeast-1
```

## Step 5: Test ALB Connection (10 minutes)

```bash
# Get ALB DNS name
aws elbv2 describe-load-balancers \
  --names ic-microservices-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text \
  --region ap-southeast-1

# Test HTTPS connection (replace with actual ALB DNS name)
curl -k https://ic-microservices-alb-123456789.ap-southeast-1.elb.amazonaws.com

# Expected: Response from API Gateway service
```

## Day 1 Completion Checklist
- ✅ Service Discovery namespace created
- ✅ ALB created with SSL certificate
- ✅ Target group created with TCP health checks
- ✅ API Gateway task definition updated with Service Connect
- ✅ API Gateway service updated and connected to ALB
- ✅ HTTPS connection working through ALB

**Next:** Day 2 - Complete Service Connect for all services + CI/CD setup
