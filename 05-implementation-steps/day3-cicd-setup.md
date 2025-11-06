# Day 3: CI/CD Pipeline Setup for Automated Deployments

**📊 Confused about GitLab integration?** → [See Day 3 Flow Diagram](./day3-flow-diagram.md)

## Prerequisites (Day 1 & 2 Complete)
- ✅ ECS services running with Service Connect
- ✅ ALB configured and healthy
- ✅ Service-to-service communication working
- ✅ All 5 microservices deployed: API Gateway, AUTH, COREv3, EMP PORTAL (Amplify), FILES

**✅ WORKING PIPELINE ARCHITECTURE:**
```
GitLab Source → CodeBuild → ECS Deploy (Staging) → Manual Approval → ECS Blue/Green (Production)
```

## Step 0: Create Internal ALB for CORE Service (CRITICAL)

### **⚠️ COMMON MISTAKE:** ALB Scheme Selection

**WRONG:** Selecting "Internet-facing" for internal service communication
**RIGHT:** Selecting "Internal" for service-to-service communication

```bash
# Create INTERNAL ALB for CORE service
aws elbv2 create-load-balancer \
  --name ic-core-internal-alb \
  --subnets subnet-096a5e3c10eef5f5c subnet-0e9a0ec15dc80197d \
  --scheme internal \  # ← CRITICAL: Must be "internal" not "internet-facing"
  --security-groups sg-YOUR_INTERNAL_SG_ID \
  --region ap-southeast-1
```

### **Console Steps:**
1. EC2 → Load Balancers → Create Application Load Balancer
2. **⚠️ IMPORTANT:** Under "Scheme" select **"Internal"**
3. Select **private subnets** (same as your ECS services)
4. Create security group allowing VPC CIDR (10.0.0.0/16)

### **Verification:**
```bash
# Verify ALB is internal
aws elbv2 describe-load-balancers \
  --names ic-core-internal-alb \
  --query 'LoadBalancers[0].{Scheme:Scheme,Subnets:AvailabilityZones[*].SubnetId}' \
  --region ap-southeast-1

# Should show: "Scheme": "internal"
```

---

## Step 1: Create CodePipeline IAM Role (10 minutes)

### **Create CodePipeline Service Role:**
```bash
# Create CodePipeline service role
aws iam create-role \
  --role-name ic-codepipeline-staging-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "codepipeline.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }' \
  --region ap-southeast-1

# Attach required policies (using basic policies that exist)
aws iam attach-role-policy \
  --role-name ic-codepipeline-staging-role \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildDeveloperAccess \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-codepipeline-staging-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-codepipeline-staging-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --region ap-southeast-1

# Add CodePipeline permissions via inline policy
aws iam put-role-policy \
  --role-name ic-codepipeline-staging-role \
  --policy-name CodePipelineInlinePolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "codepipeline:*",
          "codebuild:BatchGetBuilds",
          "codebuild:StartBuild"
        ],
        "Resource": "*"
      }
    ]
  }' \
  --region ap-southeast-1
```

## Step 2: Create CodeBuild IAM Role (10 minutes)

### **Create CodeBuild Service Role:**
```bash
# Create CodeBuild service role
aws iam create-role \
  --role-name ic-codebuild-staging-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "codebuild.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }' \
  --region ap-southeast-1

# Attach required policies
aws iam attach-role-policy \
  --role-name ic-codebuild-staging-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-codebuild-staging-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-codebuild-staging-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess \
  --region ap-southeast-1

aws iam attach-role-policy \
  --role-name ic-codebuild-staging-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --region ap-southeast-1
```

## Step 3: Create S3 Bucket for Pipeline Artifacts (5 minutes)

```bash
# Create S3 bucket for CodePipeline artifacts
aws s3 mb s3://ic-codepipeline-staging-artifacts-$(date +%s) \
  --region ap-southeast-1

# Note: Save the bucket name for pipeline configuration
export PIPELINE_BUCKET="ic-codepipeline-staging-artifacts-$(date +%s)"
echo "Pipeline bucket: $PIPELINE_BUCKET"
```

## Step 4: Create ECR Repositories (5 minutes)

**⚠️ IMPORTANT:** Create ECR repositories for all microservices first.

```bash
# Create ECR repositories for all 5 microservices
aws ecr create-repository --repository-name ic-api-gateway-image --region ap-southeast-1
aws ecr create-repository --repository-name ic-auth-image --region ap-southeast-1  
aws ecr create-repository --repository-name ic-corev3-image --region ap-southeast-1
aws ecr create-repository --repository-name ic-emp-portal-image --region ap-southeast-1
aws ecr create-repository --repository-name ic-files-image --region ap-southeast-1
```

## Step 5: Create Production ECS Services with Blue/Green (20 minutes)

**⚠️ CRITICAL:** For Blue/Green deployment, create ECS service with `CODE_DEPLOY` controller and **NO** task definition or load balancer specified.

### **API Gateway Production Service:**
```bash
aws ecs create-service \
  --cluster ic-general-services-cluster \
  --service-name ic-api-gateway-ecs-production \
  --desired-count 1 \
  --deployment-controller type=CODE_DEPLOY \
  --service-registries registryArn=arn:aws:servicediscovery:ap-southeast-1:795189341938:service/srv-a4dz6z5lonauqyay \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-096a5e3c10eef5f5c,subnet-0e9a0ec15dc80197d],securityGroups=[sg-0235a12f1b868d6dd],assignPublicIp=DISABLED}" \
  --region ap-southeast-1
```

### **CORE Production Service:**
```bash
aws ecs create-service \
  --cluster ic-general-services-cluster \
  --service-name ic-corev3-ecs-production \
  --desired-count 1 \
  --deployment-controller type=CODE_DEPLOY \
  --service-registries registryArn=arn:aws:servicediscovery:ap-southeast-1:795189341938:service/srv-a4dz6z5lonauqyay \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-096a5e3c10eef5f5c,subnet-0e9a0ec15dc80197d],securityGroups=[sg-0235a12f1b868d6dd],assignPublicIp=DISABLED}" \
  --region ap-southeast-1
```

## Step 6: Create CodeDeploy Applications (15 minutes)

### **Create CodeDeploy Applications:**
```bash
# API Gateway CodeDeploy application
aws deploy create-application \
  --application-name ic-api-gateway-cd \
  --compute-platform ECS \
  --region ap-southeast-1

# CORE CodeDeploy application  
aws deploy create-application \
  --application-name ic-core-cd \
  --compute-platform ECS \
  --region ap-southeast-1
```

### **Create Deployment Groups (Console Required):**

**⚠️ IMPORTANT:** Use AWS Console for deployment group creation with these settings:

#### **API Gateway Deployment Group:**
| Setting | Value |
|--------|-------|
| **Application** | `ic-api-gateway-cd` |
| **Deployment Group Name** | `ic-api-gateway-dg` |
| **Service** | `ic-api-gateway-ecs-production` |
| **Load Balancer** | Select your API Gateway ALB |
| **Blue Target Group** | `ic-api-gateway-blue-tg` |
| **Green Target Group** | `ic-api-gateway-green-tg` |
| **Termination Wait Time** | **10 minutes** |

#### **CORE Deployment Group:**
| Setting | Value |
|--------|-------|
| **Application** | `ic-core-cd` |
| **Deployment Group Name** | `ic-core-dg` |
| **Service** | `ic-corev3-ecs-production` |
| **Load Balancer** | Select your CORE Internal ALB |
| **Blue Target Group** | `ic-corev3-production-blue-tg` |
| **Green Target Group** | `ic-corev3-production-green-tg` |
| **Termination Wait Time** | **10 minutes** |

## Step 7: Add Required Files to GitLab Repositories (20 minutes)

**⚠️ CRITICAL:** Add these 3 files to EACH microservice GitLab repository:

### **1. buildspec.yml** (Root of repository)
```yaml
version: 0.2
env:
  variables:
    APP_NAME: ic-api-gateway-image  # Change per service
    CONTAINER_NAME: ic-api-gateway-container  # Change per service

phases:
  pre_build:
    commands:
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - REGION=$AWS_DEFAULT_REGION
      - ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"
      - IMAGE_REPO="$ECR_URI/$APP_NAME"
      - IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION:0:8}
      - aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin "$ECR_URI"

  build:
    commands:
      - docker build -t "$IMAGE_REPO:$IMAGE_TAG" -t "$IMAGE_REPO:latest" .

  post_build:
    commands:
      - docker push "$IMAGE_REPO:$IMAGE_TAG"
      - docker push "$IMAGE_REPO:latest"
      - printf '[{"name":"%s","imageUri":"%s"}]\n' "$CONTAINER_NAME" "$IMAGE_REPO:$IMAGE_TAG" > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

### **2. appspec.yml** (Root of repository)
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: taskdef.json
        LoadBalancerInfo:
          ContainerName: ic-api-gateway-container  # Change per service
          ContainerPort: 8000  # Change per service
```

### **3. taskdef.json** (Root of repository)
```json
{
  "family": "ic-api-gateway-production-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-task-role",
  "containerDefinitions": [
    {
      "name": "ic-api-gateway-container",
      "image": "PLACEHOLDER_IMAGE",
      "portMappings": [
        { 
          "containerPort": 8000, 
          "protocol": "tcp",
          "name": "api-gateway-port"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ic-api-gateway-production-task",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### **Service-Specific Values:**

#### **API Gateway:**
- `APP_NAME`: `ic-api-gateway-image`
- `CONTAINER_NAME`: `ic-api-gateway-container`
- `ContainerPort`: `8000`
- `family`: `ic-api-gateway-production-task`

#### **CORE Service:**
- `APP_NAME`: `ic-core-image`
- `CONTAINER_NAME`: `ic-core-container`
- `ContainerPort`: `8002`
- `family`: `ic-core-production-task`

## Step 8: Create CodePipeline with Blue/Green Deployment (25 minutes)

**⚠️ IMPORTANT:** Create CodeBuild projects FIRST because CodePipeline needs to reference existing projects.

### **API Gateway CodeBuild Project:**
```bash
# Create API Gateway build project
aws codebuild create-project \
  --name ic-apigateway-staging-build \
  --source '{
    "type": "GITLAB",
    "location": "YOUR_GITLAB_REPO_URL_HERE",
    "buildspec": "buildspec.yml"
  }' \
  --artifacts '{
    "type": "CODEPIPELINE"
  }' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:3.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true,
    "environmentVariables": [
      {
        "name": "AWS_DEFAULT_REGION",
        "value": "ap-southeast-1"
      },
      {
        "name": "AWS_ACCOUNT_ID",
        "value": "795189341938"
      },
      {
        "name": "IMAGE_REPO_NAME",
        "value": "ic-api-gateway-image"
      },
      {
        "name": "IMAGE_TAG",
        "value": "latest"
      },
      {
        "name": "CONTAINER_NAME",
        "value": "ic-api-gateway-container"
      },
      {
        "name": "ECS_CLUSTER_NAME",
        "value": "ic-general-services-cluster"
      },
      {
        "name": "ECS_SERVICE_NAME",
        "value": "ic-api-gateway-service-staging"
      }
    ]
  }' \
  --service-role arn:aws:iam::795189341938:role/ic-codebuild-staging-role \
  --region ap-southeast-1
```

### **AUTH Service CodeBuild Project:**
```bash
# Create AUTH build project
aws codebuild create-project \
  --name ic-auth-staging-build \
  --source '{
    "type": "GITLAB",
    "location": "YOUR_AUTH_GITLAB_REPO_URL_HERE",
    "buildspec": "buildspec.yml"
  }' \
  --artifacts '{
    "type": "CODEPIPELINE"
  }' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:3.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true,
    "environmentVariables": [
      {
        "name": "AWS_DEFAULT_REGION",
        "value": "ap-southeast-1"
      },
      {
        "name": "AWS_ACCOUNT_ID",
        "value": "795189341938"
      },
      {
        "name": "IMAGE_REPO_NAME",
        "value": "ic-auth-image"
      },
      {
        "name": "IMAGE_TAG",
        "value": "latest"
      },
      {
        "name": "CONTAINER_NAME",
        "value": "ic-auth-container"
      },
      {
        "name": "ECS_CLUSTER_NAME",
        "value": "ic-general-services-cluster"
      },
      {
        "name": "ECS_SERVICE_NAME",
        "value": "ic-auth-service-staging"
      }
    ]
  }' \
  --service-role arn:aws:iam::795189341938:role/ic-codebuild-staging-role \
  --region ap-southeast-1
```

### **CORE Service CodeBuild Project:**
```bash
# Create CORE build project
aws codebuild create-project \
  --name ic-core-staging-build \
  --source '{
    "type": "GITLAB",
    "location": "YOUR_CORE_GITLAB_REPO_URL_HERE",
    "buildspec": "buildspec.yml"
  }' \
  --artifacts '{
    "type": "CODEPIPELINE"
  }' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:3.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true,
    "environmentVariables": [
      {
        "name": "AWS_DEFAULT_REGION",
        "value": "ap-southeast-1"
      },
      {
        "name": "AWS_ACCOUNT_ID",
        "value": "795189341938"
      },
      {
        "name": "IMAGE_REPO_NAME",
        "value": "ic-core-image"
      },
      {
        "name": "IMAGE_TAG",
        "value": "latest"
      },
      {
        "name": "CONTAINER_NAME",
        "value": "ic-core-container"
      },
      {
        "name": "ECS_CLUSTER_NAME",
        "value": "ic-general-services-cluster"
      },
      {
        "name": "ECS_SERVICE_NAME",
        "value": "ic-core-service-staging"
      }
    ]
  }' \
  --service-role arn:aws:iam::795189341938:role/ic-codebuild-staging-role \
  --region ap-southeast-1
```

## Step 5: Create CodePipeline (20 minutes)

## Step 6: Add buildspec.yml to Each Repository (15 minutes)

### **Create CodePipeline with Blue/Green Deployment:**

**Pipeline Architecture:**
```
GitLab Source → CodeBuild → Deploy to Staging → Manual Approval → Deploy to Production (CodeDeploy Blue/Green)
```

#### **API Gateway Pipeline:**
```bash
aws codepipeline create-pipeline \
  --pipeline '{
    "name": "ic-api-gateway-production-pipeline",
    "roleArn": "arn:aws:iam::795189341938:role/ic-codepipeline-staging-role",
    "artifactStore": {
      "type": "S3",
      "location": "'$PIPELINE_BUCKET'"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "ThirdParty",
              "provider": "GitLab",
              "version": "1"
            },
            "configuration": {
              "Branch": "main",
              "GitLabServerUrl": "YOUR_GITLAB_URL",
              "Project": "YOUR_API_GATEWAY_PROJECT"
            },
            "outputArtifacts": [{"name": "SourceOutput"}]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "ic-apigateway-staging-build"
            },
            "inputArtifacts": [{"name": "SourceOutput"}],
            "outputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      },
      {
        "name": "Deploy-to-Staging",
        "actions": [
          {
            "name": "Deploy",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "ECS",
              "version": "1"
            },
            "configuration": {
              "ClusterName": "ic-general-services-cluster",
              "ServiceName": "ic-api-gateway-ecs-staging"
            },
            "inputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      },
      {
        "name": "Manual-Approval",
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
        "name": "Deploy-to-Production",
        "actions": [
          {
            "name": "Deploy",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeployToECS",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "ic-api-gateway-cd",
              "DeploymentGroupName": "ic-api-gateway-dg",
              "AppSpecTemplateArtifact": "BuildOutput",
              "AppSpecTemplatePath": "appspec.yml",
              "TaskDefinitionTemplateArtifact": "BuildOutput",
              "TaskDefinitionTemplatePath": "taskdef.json",
              "Image1ArtifactName": "BuildOutput",
              "Image1ContainerName": "PLACEHOLDER_IMAGE"
            },
            "inputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      }
    ]
  }' \
  --region ap-southeast-1
```

## Step 9: Test Blue/Green Deployment (15 minutes)

### **Trigger Pipeline:**
```bash
# Start pipeline manually for testing
aws codepipeline start-pipeline-execution \
  --name ic-api-gateway-production-pipeline \
  --region ap-southeast-1

# Monitor pipeline status
aws codepipeline get-pipeline-state \
  --name ic-api-gateway-production-pipeline \
  --region ap-southeast-1
```

### **Monitor Blue/Green Deployment:**
```bash
# Check CodeDeploy deployment status
aws deploy list-deployments \
  --application-name ic-api-gateway-cd \
  --region ap-southeast-1

# Get deployment details
aws deploy get-deployment \
  --deployment-id YOUR_DEPLOYMENT_ID \
  --region ap-southeast-1
```

### **Verify Blue/Green Traffic Shifting:**
```bash
# Check ALB target group weights during deployment
aws elbv2 describe-rules \
  --listener-arn YOUR_ALB_LISTENER_ARN \
  --region ap-southeast-1
```

## Step 10: Rollback Strategy

### **During Deployment (Wait Window):**
- **Console:** CodeDeploy → Deployments → Click "Stop and Roll Back"
- **CLI:** 
```bash
aws deploy stop-deployment \
  --deployment-id YOUR_DEPLOYMENT_ID \
  --auto-rollback-enabled \
  --region ap-southeast-1
```

### **After Deployment:**
- Re-deploy previous working image tag
- Use CodePipeline to deploy previous commit

---
```bash
# Create API Gateway pipeline
aws codepipeline create-pipeline \
  --pipeline '{
    "name": "ic-apigateway-staging-pipeline",
    "roleArn": "arn:aws:iam::795189341938:role/ic-codepipeline-staging-role",
    "artifactStore": {
      "type": "S3",
      "location": "'$PIPELINE_BUCKET'"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "ThirdParty",
              "provider": "GitLab",
              "version": "1"
            },
            "configuration": {
              "Branch": "main",
              "GitLabServerUrl": "YOUR_GITLAB_URL",
              "Project": "YOUR_PROJECT_NAME"
            },
            "outputArtifacts": [
              {
                "name": "SourceOutput"
              }
            ]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "ic-apigateway-staging-build"
            },
            "inputArtifacts": [
              {
                "name": "SourceOutput"
              }
            ],
            "outputArtifacts": [
              {
                "name": "BuildOutput"
              }
            ]
          }
        ]
      }
    ]
  }' \
  --region ap-southeast-1
```

## Step 6: Add buildspec.yml to Each Repository

### **Lumen Microservices buildspec.yml (add to each microservice repo):**
**File:** `buildspec.yml` (place in root of each repository)

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - echo Repository URI is $REPOSITORY_URI
      - echo Image tag is $IMAGE_TAG
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image for PHP Lumen microservice...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:latest
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo Creating imagedefinitions.json for CodePipeline...
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
      - cat imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
  name: BuildArtifact

cache:
  paths:
    - '/root/.composer/cache/**/*'
```

**Key Features:**
- ✅ **imagedefinitions.json:** CodePipeline handles ECS deployment automatically
- ✅ **Composer Cache:** Speeds up builds by caching PHP dependencies
- ✅ **Commit Hash Tagging:** Uses Git commit hash for image versioning
- ✅ **Proper Separation:** Build phase creates artifacts, CodePipeline deploys

## Step 7: Test CI/CD Pipeline (15 minutes)

### **Trigger Pipeline:**
```bash
# Start pipeline manually for testing
aws codepipeline start-pipeline-execution \
  --name ic-apigateway-staging-pipeline \
  --region ap-southeast-1

# Monitor pipeline status
aws codepipeline get-pipeline-state \
  --name ic-apigateway-staging-pipeline \
  --region ap-southeast-1
```

### **Verify Deployment:**
```bash
# Check ECS service after pipeline completes
aws ecs describe-services \
  --cluster ic-general-services-cluster \
  --services ic-api-gateway-service-staging \
  --region ap-southeast-1 \
  --query 'services[0].deployments[0].status'

# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/ic-apigateway-staging-tg/92e300ce623f18bf \
  --region ap-southeast-1
```

---

## Day 3 Completion Checklist

- ✅ CodePipeline and CodeBuild IAM roles created
- ✅ S3 bucket for pipeline artifacts created
- ✅ CodeBuild projects for all microservices created
- ✅ CodePipeline created and configured
- ✅ buildspec.yml added to repositories
- ✅ Pipeline tested and working
- ✅ Automated deployments functional

**Final Architecture Achieved:**
```
GitLab → CodePipeline → CodeBuild → ECR → ECS Fargate (with Service Connect) → ALB → Users
```

**Result:** Automated CI/CD pipeline that builds, pushes to ECR, and deploys to ECS with zero-downtime rolling updates while maintaining Service Connect communication.
