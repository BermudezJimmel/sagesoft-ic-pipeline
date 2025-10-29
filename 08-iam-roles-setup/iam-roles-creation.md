# 🔐 IAM Roles Setup Guide

## 📋 **Required IAM Roles**

Before creating task definitions, you need to create IAM roles for each service and environment.

---

## ⚡ **Step 1: Create API Gateway Staging Roles (Copy-Paste Ready)**

### **1.1 Create Execution Role**
```bash
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
```

### **1.2 Attach Execution Policies**
```bash
aws iam attach-role-policy \
  --role-name ic-apigateway-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-apigateway-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1
```

### **1.3 Create Task Role**
```bash
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
```

### **1.4 Create and Attach Task Policy**
```bash
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

aws iam attach-role-policy \
  --role-name ic-apigateway-staging-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-apigateway-staging-task-policy \
  --region ap-southeast-1
```

---

## ⚡ **Step 2: Create API Gateway Production Roles**

### **2.1 Create Production Execution Role**
```bash
aws iam create-role \
  --role-name ic-apigateway-prod-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-apigateway-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-apigateway-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1
```

### **2.2 Create Production Task Role**
```bash
aws iam create-role \
  --role-name ic-apigateway-prod-task-role \
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

aws iam create-policy \
  --policy-name ic-apigateway-prod-task-policy \
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

aws iam attach-role-policy \
  --role-name ic-apigateway-prod-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-apigateway-prod-task-policy \
  --region ap-southeast-1
```

---

## ⚡ **Step 3: Create AUTH Service Roles**

### **3.1 AUTH Staging Roles**
```bash
aws iam create-role \
  --role-name ic-auth-staging-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-auth-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-auth-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

aws iam create-role \
  --role-name ic-auth-staging-task-role \
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

aws iam create-policy \
  --policy-name ic-auth-staging-task-policy \
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
  --role-name ic-auth-staging-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-auth-staging-task-policy \
  --region ap-southeast-1
```

### **3.2 AUTH Production Roles**
```bash
aws iam create-role \
  --role-name ic-auth-prod-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-auth-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-auth-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

aws iam create-role \
  --role-name ic-auth-prod-task-role \
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

aws iam create-policy \
  --policy-name ic-auth-prod-task-policy \
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
  --role-name ic-auth-prod-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-auth-prod-task-policy \
  --region ap-southeast-1
```

---

## ⚡ **Step 4: Create CORE Service Roles**

### **4.1 CORE Staging Roles**
```bash
aws iam create-role \
  --role-name ic-core-staging-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-core-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-core-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

aws iam create-role \
  --role-name ic-core-staging-task-role \
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

aws iam create-policy \
  --policy-name ic-core-staging-task-policy \
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
  --role-name ic-core-staging-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-core-staging-task-policy \
  --region ap-southeast-1
```

### **4.2 CORE Production Roles**
```bash
aws iam create-role \
  --role-name ic-core-prod-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-core-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-core-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

aws iam create-role \
  --role-name ic-core-prod-task-role \
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

aws iam create-policy \
  --policy-name ic-core-prod-task-policy \
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
  --role-name ic-core-prod-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-core-prod-task-policy \
  --region ap-southeast-1
```

---

## ⚡ **Step 5: Create FILES Service Roles**

### **5.1 FILES Staging Roles**
```bash
aws iam create-role \
  --role-name ic-files-staging-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-files-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-files-staging-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

aws iam create-role \
  --role-name ic-files-staging-task-role \
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

aws iam create-policy \
  --policy-name ic-files-staging-task-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket",
          "secretsmanager:GetSecretValue"
        ],
        "Resource": "*"
      }
    ]
  }' \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-files-staging-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-files-staging-task-policy \
  --region ap-southeast-1
```

### **5.2 FILES Production Roles**
```bash
aws iam create-role \
  --role-name ic-files-prod-execution-role \
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

aws iam attach-role-policy \
  --role-name ic-files-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-files-prod-execution-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --region ap-southeast-1

aws iam create-role \
  --role-name ic-files-prod-task-role \
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

aws iam create-policy \
  --policy-name ic-files-prod-task-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket",
          "secretsmanager:GetSecretValue"
        ],
        "Resource": "*"
      }
    ]
  }' \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-files-prod-task-role \
  --policy-arn arn:aws:iam::795189341938:policy/ic-files-prod-task-policy \
  --region ap-southeast-1
```

---

## ✅ **Verify All Roles Created**

```bash
aws iam list-roles --query 'Roles[?starts_with(RoleName, `ic-`)].RoleName' --output table --region ap-southeast-1
```

**Expected: 16 roles total (4 services × 2 environments × 2 role types)**

---

## 🎯 **Now You Can Proceed to Day 1 Implementation!**

All IAM roles are created with proper naming convention. Use these ARNs in your task definitions:

```
ic-apigateway-staging-execution-role
ic-apigateway-staging-task-role
ic-apigateway-prod-execution-role
ic-apigateway-prod-task-role
... (and so on for auth, core, files)
```
