# Day 2: Complete Service Connect + CI/CD Setup

## Step 1: Create AUTH Service Task Definition and Service (25 minutes)

### **1.1 Create AUTH Task Definition File**

Create file: `ic-auth-staging-task-definition.json`

```json
{
  "family": "ic-auth-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-auth-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-auth-staging-task-role",
  "containerDefinitions": [
    {
      "name": "ic-auth-container",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/ic-auth-image:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "ic-auth-port",
          "containerPort": 8001,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        {
          "name": "APP_NAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:APP_NAME::"
        },
        {
          "name": "APP_ENV",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:APP_ENV::"
        },
        {
          "name": "APP_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:APP_KEY::"
        },
        {
          "name": "APP_DEBUG",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:APP_DEBUG::"
        },
        {
          "name": "APP_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:APP_URL::"
        },
        {
          "name": "APP_TIMEZONE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:APP_TIMEZONE::"
        },
        {
          "name": "LOG_CHANNEL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:LOG_CHANNEL::"
        },
        {
          "name": "LOG_SLACK_WEBHOOK_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:LOG_SLACK_WEBHOOK_URL::"
        },
        {
          "name": "DB_CONNECTION",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_CONNECTION::"
        },
        {
          "name": "DB_HOST",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_HOST::"
        },
        {
          "name": "DB_PORT",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_PORT::"
        },
        {
          "name": "DB_DATABASE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_DATABASE::"
        },
        {
          "name": "DB_USERNAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_USERNAME::"
        },
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_PASSWORD::"
        },
        {
          "name": "DB_TIMEZONE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:DB_TIMEZONE::"
        },
        {
          "name": "PROFILE_PICS_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:PROFILE_PICS_URL::"
        },
        {
          "name": "CACHE_DRIVER",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:CACHE_DRIVER::"
        },
        {
          "name": "QUEUE_CONNECTION",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:QUEUE_CONNECTION::"
        },
        {
          "name": "ACCEPTED_SECRETS",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:ACCEPTED_SECRETS::"
        },
        {
          "name": "FILES_MICROSERVICE_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:FILES_MICROSERVICE_URL::"
        },
        {
          "name": "FILES_MICROSERVICE_SECRET",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-auth-staging-secrets-WYNXVe:FILES_MICROSERVICE_SECRET::"
        }
      ],
      "environment": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-auth-staging-logs",
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

### **1.2 Register AUTH Task Definition**
```bash
aws ecs register-task-definition \
  --cli-input-json file://ic-auth-staging-task-definition.json \
  --region ap-southeast-1
```

### **1.3 Create AUTH ECS Service with Service Connect**
```bash
# CREATE AUTH service (first time - not update)
aws ecs create-service \
  --cluster ic-general-services-cluster \
  --service-name ic-auth-service-staging \
  --task-definition ic-auth-staging-task \
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
    "namespace": "arn:aws:servicediscovery:ap-southeast-1:795189341938:namespace/ns-xbe5ptxbnzf3cu2z",
    "services": [
      {
        "portName": "ic-auth-port",
        "discoveryName": "auth-service",
        "clientAliases": [
          {
            "port": 8001,
            "dnsName": "auth-service.local"
          }
        ]
      }
    ]
  }' \
  --region ap-southeast-1

# Verify AUTH service creation
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-auth-service-staging \
  --region ap-southeast-1

# If you need to update Service Connect configuration later:
aws ecs update-service \
  --cluster ic-general-services-cluster \
  --service ic-auth-service-staging \
  --task-definition ic-auth-staging-task \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "arn:aws:servicediscovery:ap-southeast-1:795189341938:namespace/ns-xbe5ptxbnzf3cu2z",
    "services": [
      {
        "portName": "ic-auth-port",
        "discoveryName": "auth-service",
        "clientAliases": [
          {
            "port": 8001,
            "dnsName": "auth-service.local"
          }
        ]
      }
    ]
  }' \
  --region ap-southeast-1
```

**⚠️ Important:** Always use the correct port name from task definition (`ic-auth-port`, not `auth-port`).

---

## Step 2: Create CORE Service Task Definition and Service (25 minutes)

### **2.1 Create CORE Task Definition File**

Create file: `ic-core-staging-task-definition.json`

```json
{
  "family": "ic-core-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-core-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-core-staging-task-role",
  "containerDefinitions": [
    {
      "name": "ic-core-container",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/ic-core-image:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "ic-core-port",
          "containerPort": 8002,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        {
          "name": "APP_NAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-staging-secrets-XXXXXX:APP_NAME::"
        }
      ],
      "environment": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-core-staging-logs",
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

**Note:** Replace `ic-core-staging-secrets-XXXXXX` with your actual CORE secrets ARN.

### **2.2 Register CORE Task Definition**
```bash
aws ecs register-task-definition \
  --cli-input-json file://ic-core-staging-task-definition.json \
  --region ap-southeast-1
```

### **2.3 Create CORE ECS Service with Service Connect**
```bash
# CREATE CORE service
aws ecs create-service \
  --cluster ic-general-services-cluster \
  --service-name ic-core-service-staging \
  --task-definition ic-core-staging-task \
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
    "namespace": "arn:aws:servicediscovery:ap-southeast-1:795189341938:namespace/ns-xbe5ptxbnzf3cu2z",
    "services": [
      {
        "portName": "ic-core-port",
        "discoveryName": "core-service",
        "clientAliases": [
          {
            "port": 8002,
            "dnsName": "core-service.local"
          }
        ]
      }
    ]
  }' \
  --region ap-southeast-1

# Verify CORE service creation
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-core-service-staging \
  --region ap-southeast-1
```

---

## Step 3: Create FILES Service Task Definition and Service (25 minutes)

### **3.1 Create FILES Task Definition File**

Create file: `ic-files-staging-task-definition.json`

```json
{
  "family": "ic-files-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-files-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-files-staging-task-role",
  "containerDefinitions": [
    {
      "name": "ic-files-container",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/ic-files-image:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "ic-files-port",
          "containerPort": 8003,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        {
          "name": "APP_NAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-files-staging-secrets-XXXXXX:APP_NAME::"
        }
      ],
      "environment": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-files-staging-logs",
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

**Note:** Replace `ic-files-staging-secrets-XXXXXX` with your actual FILES secrets ARN.

### **3.2 Register FILES Task Definition**
```bash
aws ecs register-task-definition \
  --cli-input-json file://ic-files-staging-task-definition.json \
  --region ap-southeast-1
```

### **3.3 Create FILES ECS Service with Service Connect**
```bash
# CREATE FILES service
aws ecs create-service \
  --cluster ic-general-services-cluster \
  --service-name ic-files-service-staging \
  --task-definition ic-files-staging-task \
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
    "namespace": "arn:aws:servicediscovery:ap-southeast-1:795189341938:namespace/ns-xbe5ptxbnzf3cu2z",
    "services": [
      {
        "portName": "ic-files-port",
        "discoveryName": "files-service",
        "clientAliases": [
          {
            "port": 8003,
            "dnsName": "files-service.local"
          }
        ]
      }
    ]
  }' \
  --region ap-southeast-1

# Verify FILES service creation
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-files-service-staging \
  --region ap-southeast-1
```

---

## Step 4: Test Service Connect Communication (15 minutes)

### **4.1 Test Service Discovery**
```bash
# Test if all services are discoverable
aws servicediscovery list-services \
  --filters Name=NAMESPACE_ID,Values=ns-xbe5ptxbnzf3cu2z \
  --region ap-southeast-1

# Check all ECS services status
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-api-gateway-service-staging ic-auth-service-staging ic-core-service-staging ic-files-service-staging \
  --query 'services[].{Name:serviceName,Status:status,Running:runningCount,Desired:desiredCount}' \
  --region ap-southeast-1
```

### **4.2 Test ALB Connection**
```bash
# Get ALB DNS name and test
aws elbv2 describe-load-balancers \
  --names ic-apigateway-staging-lb \
  --query 'LoadBalancers[0].DNSName' \
  --output text \
  --region ap-southeast-1

# Test HTTPS connection
curl -k https://YOUR_ALB_DNS_NAME
```

---

## Step 5: Database Updates (CRITICAL)

**Update database with Service Connect URLs - see:** [Database Updates Guide](../09-database-updates/service-connect-database-migration.md)

```sql
-- Update microservice URLs in database
UPDATE microservices_config SET 
  url = 'http://auth-service.local:8001' 
WHERE service_name = 'auth';

UPDATE microservices_config SET 
  url = 'http://core-service.local:8002' 
WHERE service_name = 'core';

UPDATE microservices_config SET 
  url = 'http://files-service.local:8003' 
WHERE service_name = 'files';
```

---

## Day 2 Completion Checklist

- ✅ AUTH service created with Service Connect (`auth-service.local:8001`)
- ✅ CORE service created with Service Connect (`core-service.local:8002`)
- ✅ FILES service created with Service Connect (`files-service.local:8003`)
- ✅ All services running and healthy
- ✅ Service Connect communication working
- ✅ ALB serving API Gateway over HTTPS
- ✅ Database updated with Service Connect URLs

**Final Architecture Achieved:**
```
EMP (Amplify) → ALB (HTTPS) → API Gateway (8000) → Service Connect → {AUTH (8001), CORE (8002), FILES (8003)} → Multi-AZ RDS
```

**Next:** Set up CI/CD pipelines for automated deployments
