# Day 2: Complete Service Connect + CI/CD Setup

## Prerequisites: CloudWatch Logs Setup (5 minutes) - **REQUIRED FIRST**

**⚠️ CRITICAL:** Create log groups and add CloudWatch permissions before creating services to avoid task failures.

### **Create Log Groups for All Services:**
```bash
# Create log groups for all microservices
aws logs create-log-group \
  --log-group-name /ecs/ic-auth-staging-logs \
  --region ap-southeast-1

aws logs create-log-group \
  --log-group-name /ecs/ic-core-staging-logs \
  --region ap-southeast-1

aws logs create-log-group \
  --log-group-name /ecs/ic-files-staging-logs \
  --region ap-southeast-1
```

### **Add CloudWatch Permissions to All Execution Roles:**
```bash
# Add CloudWatch permissions to AUTH execution role
aws iam attach-role-policy \
  --role-name ic-auth-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
  --region ap-southeast-1

# Add CloudWatch permissions to CORE execution role
aws iam attach-role-policy \
  --role-name ic-core-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
  --region ap-southeast-1

# Add CloudWatch permissions to FILES execution role
aws iam attach-role-policy \
  --role-name ic-files-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
  --region ap-southeast-1
```

**✅ Complete these prerequisites before proceeding with service creation!**

---

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
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/ic-core-v3-image:latest",
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
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:APP_NAME::"
        },
        {
          "name": "APP_ENV",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:APP_ENV::"
        },
        {
          "name": "APP_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:APP_KEY::"
        },
        {
          "name": "APP_DEBUG",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:APP_DEBUG::"
        },
        {
          "name": "APP_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:APP_URL::"
        },
        {
          "name": "APP_TIMEZONE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:APP_TIMEZONE::"
        },
        {
          "name": "LOG_CHANNEL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:LOG_CHANNEL::"
        },
        {
          "name": "LOG_SLACK_WEBHOOK_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:LOG_SLACK_WEBHOOK_URL::"
        },
        {
          "name": "DB_CONNECTION",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_CONNECTION::"
        },
        {
          "name": "DB_HOST",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_HOST::"
        },
        {
          "name": "DB_PORT",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_PORT::"
        },
        {
          "name": "DB_DATABASE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_DATABASE::"
        },
        {
          "name": "DB_USERNAME",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_USERNAME::"
        },
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_PASSWORD::"
        },
        {
          "name": "DB_TIMEZONE",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:DB_TIMEZONE::"
        },
        {
          "name": "PROFILE_PICS_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:PROFILE_PICS_URL::"
        },
        {
          "name": "ACCEPTED_SECRETS",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:ACCEPTED_SECRETS::"
        },
        {
          "name": "FILES_MICROSERVICE_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:FILES_MICROSERVICE_URL::"
        },
        {
          "name": "FILES_MICROSERVICE_SECRET",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-core-v3-staging-secrets-O9MoQK:FILES_MICROSERVICE_SECRET::"
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

---

## 🔧 Troubleshooting Service Connect DNS Resolution

### **Issue: "Could not resolve host: core-service.local"**

**Symptoms:**
- Services can communicate using private IPs but not Service Connect DNS names
- Error: `cURL error 6: Could not resolve host: core-service.local`
- Private IP works: `http://10.0.x.x:8002` ✅
- Service Connect DNS fails: `http://core-service.local:8002` ❌

### **Root Cause Analysis Steps:**

#### **Step 1: Verify Service Discovery Registration**
```bash
# Check if services are registered in Service Discovery
aws servicediscovery list-services \
  --filters Name=NAMESPACE_ID,Values=ns-xbe5ptxbnzf3cu2z \
  --query 'Services[].{Name:Name,Type:Type}' \
  --region ap-southeast-1
```
**Expected Output:** Should show `api-gateway`, `auth-service`, `core-service`

#### **Step 2: Check ECS Cluster Service Connect Configuration**
```bash
# Verify cluster has Service Connect enabled
aws ecs describe-clusters \
  --clusters ic-general-services-cluster \
  --include CONFIGURATIONS \
  --region ap-southeast-1
```
**Look for:** `serviceConnectDefaults` configuration

#### **Step 3: Verify Service Connect Configuration on Services**
```bash
# Check API Gateway service configuration
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-apigateway-staging-service \
  --region ap-southeast-1 \
  --query 'services[0].serviceConnectConfiguration'

# Check CORE service configuration  
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-core-staging-service \
  --region ap-southeast-1 \
  --query 'services[0].serviceConnectConfiguration'
```

### **Solution Steps:**

#### **Fix 1: Update Cluster Configuration (if needed)**
```bash
# Enable Service Connect on cluster level
aws ecs put-cluster-capacity-providers \
  --cluster ic-general-services-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --region ap-southeast-1
```

#### **Fix 2: Recreate Services with Proper Service Connect**
If Service Connect configuration shows `null`, recreate the services:

```bash
# Delete and recreate API Gateway service
aws ecs delete-service \
  --cluster ic-general-services-cluster \
  --service ic-apigateway-staging-service \
  --force \
  --region ap-southeast-1

# Wait for deletion, then recreate with proper Service Connect configuration
# (Use the service creation commands from Step 2 above)
```

#### **Fix 3: Verify Task Definition Network Mode**
Ensure task definitions use `awsvpc` network mode:
```json
{
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"]
}
```

#### **Fix 4: Check Security Group Rules**
```bash
# Verify security groups allow internal communication
aws ec2 describe-security-groups \
  --group-ids sg-your-security-group-id \
  --region ap-southeast-1
```
**Required:** Allow inbound traffic on ports 8001, 8002, 8003 from same security group

### **Verification Commands:**

#### **Test Service Connect Resolution:**
```bash
# From within API Gateway container, test DNS resolution
nslookup core-service.local
nslookup auth-service.local

# Test HTTP connectivity
curl -v http://core-service.local:8002/health
curl -v http://auth-service.local:8001/health
```

#### **Check CloudWatch Logs:**
```bash
# Check API Gateway logs for connection errors
aws logs filter-log-events \
  --log-group-name /ecs/ic-apigateway-staging-logs \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --region ap-southeast-1
```

### **Expected Resolution Timeline:**
- Service Discovery registration: 1-2 minutes
- DNS propagation: 2-3 minutes  
- Service Connect activation: 3-5 minutes

**✅ Success Indicator:** API Gateway can successfully call `http://core-service.local:8002/people` without DNS resolution errors.

---

**Next:** Set up CI/CD pipelines for automated deployments
