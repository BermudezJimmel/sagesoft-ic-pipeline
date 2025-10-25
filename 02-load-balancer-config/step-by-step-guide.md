# Step-by-Step Implementation Guide

## Day 1: Load Balancer + Service Connect Setup

### Step 1: Create Application Load Balancer
```bash
# Create ALB for API Gateway only
aws elbv2 create-load-balancer \
  --name ic-microservices-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx \
  --scheme internet-facing \
  --type application
```

### Step 2: Create Target Group for API Gateway
```bash
# Target group for API Gateway service
aws elbv2 create-target-group \
  --name api-gateway-tg \
  --protocol HTTP \
  --port 8000 \
  --vpc-id vpc-xxx \
  --target-type ip \
  --health-check-path /health
```

### Step 3: ECS Service Connect Configuration
```json
{
  "serviceConnectConfiguration": {
    "enabled": true,
    "namespace": "ic-microservices",
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
  }
}
```

## Day 2: Updated Task Definitions with Secrets

### Enhanced Task Definition Template
```json
{
  "family": "api-gateway-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
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
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:password::"
        }
      ],
      "environment": [
        {
          "name": "CORE_SERVICE_URL",
          "value": "http://core-service.local:8001"
        },
        {
          "name": "AUTH_SERVICE_URL", 
          "value": "http://auth-service.local:8002"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/api-gateway",
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

## Key Changes from Your Current Setup
1. **Added Service Connect** - No more internal ALB needed
2. **Secrets Integration** - Database credentials from Secrets Manager
3. **Service Discovery** - Services communicate via DNS names
4. **Port Naming** - Required for Service Connect
