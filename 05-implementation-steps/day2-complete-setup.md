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
```

---

## Step 2: Update CORE and FILES Services (20 minutes)

**CORE Service Task Definition:** `ic-core-staging-task-definition.json`
```json
{
  "family": "ic-core-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-core-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-core-staging-task-role",
  "containerDefinitions": [
    {
      "name": "core",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/core:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "core-port",
          "containerPort": 8002,
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
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-core-staging",
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

**FILES Service Task Definition:** `ic-files-staging-task-definition.json`
```json
{
  "family": "ic-files-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-files-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-files-staging-task-role",
  "containerDefinitions": [
    {
      "name": "files",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/files:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "files-port",
          "containerPort": 8003,
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
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-files-staging",
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

**Register and Update Services:**
```bash
# Register CORE task definition
aws ecs register-task-definition \
  --cli-input-json file://ic-core-staging-task-definition.json \
  --region ap-southeast-1

# Update CORE service
aws ecs update-service \
  --cluster ic-general-services-cluster \
  --service REPLACE_WITH_CORE_SERVICE_NAME \
  --task-definition ic-core-staging-task \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "arn:aws:servicediscovery:ap-southeast-1:795189341938:namespace/ns-xbe5ptxbnzf3cu2z",
    "services": [
      {
        "portName": "core-port",
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

# Register FILES task definition
aws ecs register-task-definition \
  --cli-input-json file://ic-files-staging-task-definition.json \
  --region ap-southeast-1

# Update FILES service
aws ecs update-service \
  --cluster ic-general-services-cluster \
  --service REPLACE_WITH_FILES_SERVICE_NAME \
  --task-definition ic-files-staging-task \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "arn:aws:servicediscovery:ap-southeast-1:795189341938:namespace/ns-xbe5ptxbnzf3cu2z",
    "services": [
      {
        "portName": "files-port",
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
```
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/auth",
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

```bash
# Register and update AUTH service
aws ecs register-task-definition \
  --cli-input-json file://auth-task-definition.json \
  --region ap-southeast-1

aws ecs update-service \
  --cluster REPLACE_WITH_YOUR_CLUSTER_NAME \
  --service REPLACE_WITH_AUTH_SERVICE_NAME \
  --task-definition auth-task \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "REPLACE_WITH_NAMESPACE_ID",
    "services": [
      {
        "portName": "auth-port",
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

## Step 2: Update CORE and FILES Services (20 minutes)

**CORE Service:**
```json
{
  "family": "core-task",
  "containerDefinitions": [
    {
      "name": "core",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/core:latest",
      "portMappings": [
        {
          "name": "core-port",
          "containerPort": 8002,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

**FILES Service:**
```json
{
  "family": "files-task",
  "containerDefinitions": [
    {
      "name": "files",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/files:latest",
      "portMappings": [
        {
          "name": "files-port",
          "containerPort": 8003,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

## Step 3: Create CodePipeline for Each Service (30 minutes)

### Create S3 Bucket for Artifacts
```bash
aws s3 mb s3://ic-microservices-codepipeline-artifacts-RANDOM_SUFFIX --region ap-southeast-1
```

### Create CodeBuild Project Template
```bash
aws codebuild create-project \
  --name ic-microservices-build-api-gateway \
  --source '{
    "type": "CODEPIPELINE",
    "buildspec": "buildspec.yml"
  }' \
  --artifacts '{
    "type": "CODEPIPELINE"
  }' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:3.0",
    "computeType": "BUILD_GENERAL1_SMALL",
    "privilegedMode": true,
    "environmentVariables": [
      {
        "name": "IMAGE_REPO_NAME",
        "value": "api-gateway"
      },
      {
        "name": "CONTAINER_NAME",
        "value": "api-gateway"
      }
    ]
  }' \
  --service-role arn:aws:iam::795189341938:role/CodeBuildServiceRole \
  --region ap-southeast-1
```

### Create CodePipeline Template
Save as `api-gateway-pipeline.json`:
```json
{
  "pipeline": {
    "name": "ic-microservices-api-gateway-pipeline",
    "roleArn": "arn:aws:iam::795189341938:role/CodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "ic-microservices-codepipeline-artifacts-RANDOM_SUFFIX"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "ThirdParty",
              "provider": "GitLab",
              "version": "1"
            },
            "configuration": {
              "GitLabServerUrl": "https://gitlab.com",
              "RepositoryName": "CLIENT_TO_REPLACE/api-gateway-repo",
              "BranchName": "main"
            },
            "outputArtifacts": [{"name": "SourceOutput"}]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAction",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "ic-microservices-build-api-gateway"
            },
            "inputArtifacts": [{"name": "SourceOutput"}],
            "outputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      },
      {
        "name": "Deploy-Staging",
        "actions": [
          {
            "name": "DeployStaging",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "ECS",
              "version": "1"
            },
            "configuration": {
              "ClusterName": "REPLACE_WITH_CLUSTER_NAME",
              "ServiceName": "api-gateway-staging",
              "FileName": "imagedefinitions.json"
            },
            "inputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      },
      {
        "name": "Approval",
        "actions": [
          {
            "name": "ManualApproval",
            "actionTypeId": {
              "category": "Approval",
              "owner": "AWS",
              "provider": "Manual",
              "version": "1"
            },
            "configuration": {
              "CustomData": "Please review staging deployment and approve for production"
            }
          }
        ]
      },
      {
        "name": "Deploy-Production",
        "actions": [
          {
            "name": "DeployProduction",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "ECS",
              "version": "1"
            },
            "configuration": {
              "ClusterName": "REPLACE_WITH_CLUSTER_NAME",
              "ServiceName": "api-gateway-prod",
              "FileName": "imagedefinitions.json"
            },
            "inputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      }
    ]
  }
}
```

Create pipeline:
```bash
aws codepipeline create-pipeline \
  --cli-input-json file://api-gateway-pipeline.json \
  --region ap-southeast-1
```

## Step 4: Test Complete Setup (20 minutes)

```bash
# Test Service Connect communication
# SSH into API Gateway container and test:
curl http://auth-service.local:8001/health
curl http://core-service.local:8002/health
curl http://files-service.local:8003/health

# Test ALB to API Gateway
curl -k https://YOUR_ALB_DNS_NAME

# Test CodePipeline (trigger manually first time)
aws codepipeline start-pipeline-execution \
  --name ic-microservices-api-gateway-pipeline \
  --region ap-southeast-1
```

## Day 2 Completion Checklist
- ✅ All 4 services updated with Service Connect
- ✅ Services can communicate via .local DNS names
- ✅ CodeBuild projects created for all services
- ✅ CodePipeline created with staging/production stages
- ✅ Manual approval gate configured
- ✅ End-to-end pipeline test successful

## Final Architecture Achieved
```
EMP (Amplify) → ALB (HTTPS) → API Gateway (8000) → Service Connect → {AUTH (8001), CORE (8002), FILES (8003)} → Multi-AZ RDS
                                                                                                                    ↑
GitLab Push → CodePipeline → CodeBuild → ECR → ECS Staging → Manual Approval → ECS Production ──────────────────┘
```

**Implementation Complete! 🎉**
