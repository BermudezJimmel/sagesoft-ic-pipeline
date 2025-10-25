# Day 2: Complete Service Connect + CI/CD Setup

## Step 1: Update AUTH Service Task Definition (15 minutes)

Create file: `auth-task-definition.json`

```json
{
  "family": "auth-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "auth",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/auth:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "auth-port",
          "containerPort": 8001,
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
          "value": "REPLACE_WITH_SCHEMA_NAME"
        },
        {
          "name": "CORE_SERVICE_URL",
          "value": "http://core-service.local:8002"
        }
      ],
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
