# Day 3: CI/CD Pipeline Setup for Automated Deployments

## Prerequisites (Day 1 & 2 Complete)
- ✅ ECS services running with Service Connect
- ✅ ALB configured and healthy
- ✅ Service-to-service communication working
- ✅ All microservices deployed manually

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

## Step 4: Create CodeBuild Projects (15 minutes)

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

### **API Gateway Pipeline:**
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
      - echo Updating ECS service...
      - |
        aws ecs update-service \
          --cluster $ECS_CLUSTER_NAME \
          --service $ECS_SERVICE_NAME \
          --force-new-deployment \
          --region $AWS_DEFAULT_REGION
      - echo Waiting for deployment to complete...
      - |
        aws ecs wait services-stable \
          --cluster $ECS_CLUSTER_NAME \
          --services $ECS_SERVICE_NAME \
          --region $AWS_DEFAULT_REGION
      - echo Deployment completed successfully

artifacts:
  files:
    - '**/*'
  name: BuildArtifact

cache:
  paths:
    - '/root/.composer/cache/**/*'
```

**Key Features:**
- ✅ **Composer Cache:** Speeds up builds by caching PHP dependencies
- ✅ **Commit Hash Tagging:** Uses Git commit hash for image versioning
- ✅ **Zero Downtime:** ECS rolling deployment with wait for stability
- ✅ **Service Connect Preserved:** Maintains Service Connect during deployments

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
