# 🔐 IAM Roles Setup Guide

## 📋 **Required IAM Roles**

Before creating task definitions, you need to create IAM roles for each service and environment.

---

## 🎯 **Naming Convention**

```
Service: ic-{service}-{environment}-{role-type}

Examples:
- ic-apigateway-staging-execution-role
- ic-apigateway-staging-task-role
- ic-apigateway-prod-execution-role
- ic-apigateway-prod-task-role
```

---

## ⚡ **Step 1: Create API Gateway Roles (Staging)**

### **Execution Role (Required for ECS to pull images, logs, secrets)**
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

# Attach ECS task execution policy
aws iam attach-role-policy \
  --role-name ic-apigateway-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

# Attach Secrets Manager access
aws iam attach-role-policy \
  --role-name ic-apigateway-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1
```

### **Task Role (Required for application to access AWS services)**
```bash
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

# Create custom policy for application needs
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
          "s3:DeleteObject",
          "rds:DescribeDBInstances",
          "secretsmanager:GetSecretValue"
        ],
        "Resource": "*"
      }
    ]
  }' \
  --region ap-southeast-1

# Attach custom policy to task role
aws iam attach-role-policy \
  --role-name ic-apigateway-staging-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-apigateway-staging-task-policy \
  --region ap-southeast-1
```

---

## 🔄 **Step 2: Create All Service Roles**

### **Quick Script to Create All Roles:**
```bash
#!/bin/bash

SERVICES=("apigateway" "auth" "core" "files")
ENVIRONMENTS=("staging" "prod")
ACCOUNT_ID="795189341938"
REGION="ap-southeast-1"

for service in "${SERVICES[@]}"; do
  for env in "${ENVIRONMENTS[@]}"; do
    echo "Creating roles for ic-${service}-${env}..."
    
    # Create execution role
    aws iam create-role \
      --role-name ic-${service}-${env}-execution-role \
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
      --region ${REGION}
    
    # Attach execution policies
    aws iam attach-role-policy \
      --role-name ic-${service}-${env}-execution-role \
      --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
      --region ${REGION}
    
    aws iam attach-role-policy \
      --role-name ic-${service}-${env}-execution-role \
      --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
      --region ${REGION}
    
    # Create task role
    aws iam create-role \
      --role-name ic-${service}-${env}-task-role \
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
      --region ${REGION}
    
    # Create and attach task policy
    aws iam create-policy \
      --policy-name ic-${service}-${env}-task-policy \
      --policy-document '{
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": [
              "s3:GetObject",
              "s3:PutObject",
              "s3:DeleteObject",
              "rds:DescribeDBInstances",
              "secretsmanager:GetSecretValue"
            ],
            "Resource": "*"
          }
        ]
      }' \
      --region ${REGION} 2>/dev/null || echo "Policy ic-${service}-${env}-task-policy already exists"
    
    aws iam attach-role-policy \
      --role-name ic-${service}-${env}-task-role \
      --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ic-${service}-${env}-task-policy \
      --region ${REGION}
    
    echo "✅ Completed ic-${service}-${env} roles"
  done
done

echo "🎉 All IAM roles created successfully!"
```

---

## 📋 **Step 3: Verify Roles Created**

```bash
# List all IC microservices roles
aws iam list-roles --query 'Roles[?starts_with(RoleName, `ic-`)].RoleName' --output table --region ap-southeast-1
```

**Expected Output:**
```
ic-apigateway-staging-execution-role
ic-apigateway-staging-task-role
ic-apigateway-prod-execution-role
ic-apigateway-prod-task-role
ic-auth-staging-execution-role
ic-auth-staging-task-role
ic-auth-prod-execution-role
ic-auth-prod-task-role
ic-core-staging-execution-role
ic-core-staging-task-role
ic-core-prod-execution-role
ic-core-prod-task-role
ic-files-staging-execution-role
ic-files-staging-task-role
ic-files-prod-execution-role
ic-files-prod-task-role
```

---

## ✅ **Updated Task Definition ARNs**

### **API Gateway Staging:**
```json
{
  "family": "ic-apigateway-staging-task",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-task-role"
}
```

### **API Gateway Production:**
```json
{
  "family": "ic-apigateway-prod-task",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-prod-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-prod-task-role"
}
```

---

## 🚨 **Important Notes**

1. **Run this BEFORE creating task definitions**
2. **Each service needs separate roles** for proper security isolation
3. **Staging and production use separate roles** for environment isolation
4. **Execution role** = ECS infrastructure permissions
5. **Task role** = Application permissions (what your code can access)

---

## 🔧 **Troubleshooting**

| Error | Solution |
|-------|----------|
| Role already exists | Skip creation, use existing role |
| Policy already exists | Skip policy creation, attach existing policy |
| Access denied | Check your AWS credentials have IAM permissions |
| Invalid JSON | Copy-paste the exact JSON from above |

**✅ Complete this step before proceeding with task definitions!**
