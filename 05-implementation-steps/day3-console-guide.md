# Day 3: CI/CD Pipeline Setup (AWS Console Guide)

## Prerequisites (Day 1 & 2 Complete)
- ✅ ECS services running with Service Connect
- ✅ ALB configured and healthy
- ✅ Service-to-service communication working

## Step 1: Create IAM Roles (15 minutes)

### **1.1 Create CodePipeline Service Role**

1. **Go to IAM Console** → Roles → Create role
2. **Select trusted entity:** AWS service
3. **Use case:** CodePipeline → Next
4. **Role name:** `ic-codepipeline-staging-role`
5. **Attach policies:**
   - `AWSCodeBuildDeveloperAccess`
   - `AmazonECS_FullAccess`
   - `AmazonS3FullAccess`
6. **Create role**

### **1.2 Add CodePipeline Inline Policy**

1. **Open role:** `ic-codepipeline-staging-role`
2. **Add permissions** → Create inline policy
3. **JSON tab:**
```json
{
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
}
```
4. **Policy name:** `CodePipelineInlinePolicy`
5. **Create policy**

### **1.3 Create CodeBuild Service Role**

1. **Go to IAM Console** → Roles → Create role
2. **Select trusted entity:** AWS service
3. **Use case:** CodeBuild → Next
4. **Role name:** `ic-codebuild-staging-role`
5. **Attach policies:**
   - `CloudWatchLogsFullAccess`
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonECS_FullAccess`
   - `AmazonS3FullAccess`
6. **Create role**

## Step 2: Create S3 Bucket for Artifacts (5 minutes)

1. **Go to S3 Console** → Create bucket
2. **Bucket name:** `ic-codepipeline-staging-artifacts-YYYYMMDD` (use today's date)
3. **Region:** Asia Pacific (Singapore) ap-southeast-1
4. **Block all public access:** ✅ Enabled
5. **Create bucket**
6. **📝 Note the bucket name** for pipeline configuration

## Step 3: Create CodeBuild Projects (45 minutes)

### **3.1 API Gateway CodeBuild Project**

1. **Go to CodeBuild Console** → Create build project
2. **Project name:** `ic-apigateway-staging-build`

**Source:**
- **Source provider:** GitLab
- **Repository URL:** `YOUR_API_GATEWAY_GITLAB_REPO_URL`
- **Buildspec:** Use a buildspec file
- **Buildspec name:** `buildspec.yml`

**Environment:**
- **Environment image:** Managed image
- **Operating system:** Amazon Linux 2
- **Runtime:** Standard
- **Image:** aws/codebuild/amazonlinux2-x86_64-standard:3.0
- **Privileged:** ✅ Enable (for Docker)
- **Service role:** Existing service role
- **Role ARN:** `arn:aws:iam::795189341938:role/ic-codebuild-staging-role`

**Environment variables:**
| Name | Value |
|------|-------|
| `AWS_DEFAULT_REGION` | `ap-southeast-1` |
| `AWS_ACCOUNT_ID` | `795189341938` |
| `IMAGE_REPO_NAME` | `ic-api-gateway-image` |
| `IMAGE_TAG` | `latest` |
| `CONTAINER_NAME` | `ic-api-gateway-container` |
| `ECS_CLUSTER_NAME` | `ic-general-services-cluster` |
| `ECS_SERVICE_NAME` | `ic-api-gateway-service-staging` |

3. **Create build project**

### **3.2 AUTH Service CodeBuild Project**

1. **Create build project:** `ic-auth-staging-build`
2. **Source:** Your AUTH GitLab repository URL
3. **Environment:** Same as API Gateway
4. **Environment variables:**

| Name | Value |
|------|-------|
| `AWS_DEFAULT_REGION` | `ap-southeast-1` |
| `AWS_ACCOUNT_ID` | `795189341938` |
| `IMAGE_REPO_NAME` | `ic-auth-image` |
| `IMAGE_TAG` | `latest` |
| `CONTAINER_NAME` | `ic-auth-container` |
| `ECS_CLUSTER_NAME` | `ic-general-services-cluster` |
| `ECS_SERVICE_NAME` | `ic-auth-service-staging` |

### **3.3 CORE Service CodeBuild Project**

1. **Create build project:** `ic-core-staging-build`
2. **Source:** Your CORE GitLab repository URL
3. **Environment:** Same as API Gateway
4. **Environment variables:**

| Name | Value |
|------|-------|
| `AWS_DEFAULT_REGION` | `ap-southeast-1` |
| `AWS_ACCOUNT_ID` | `795189341938` |
| `IMAGE_REPO_NAME` | `ic-core-image` |
| `IMAGE_TAG` | `latest` |
| `CONTAINER_NAME` | `ic-core-container` |
| `ECS_CLUSTER_NAME` | `ic-general-services-cluster` |
| `ECS_SERVICE_NAME` | `ic-core-service-staging` |

## Step 4: Create CodePipeline (30 minutes)

### **4.1 API Gateway Pipeline**

1. **Go to CodePipeline Console** → Create pipeline
2. **Pipeline name:** `ic-apigateway-staging-pipeline`
3. **Service role:** Existing service role
4. **Role ARN:** `arn:aws:iam::795189341938:role/ic-codepipeline-staging-role`
5. **Artifact store:** Custom location
6. **Bucket:** Select your S3 bucket created in Step 2

**Source Stage:**
- **Source provider:** GitLab
- **Connection:** Create new connection (if first time)
- **Repository name:** Your API Gateway repository
- **Branch name:** `main`
- **Output artifacts:** `SourceOutput`

**Build Stage:**
- **Build provider:** AWS CodeBuild
- **Project name:** `ic-apigateway-staging-build`
- **Input artifacts:** `SourceOutput`
- **Output artifacts:** `BuildOutput`

**Deploy Stage:**
- **Deploy provider:** Amazon ECS
- **Cluster name:** `ic-general-services-cluster`
- **Service name:** `ic-api-gateway-service-staging`
- **Image definitions file:** `imagedefinitions.json`
- **Input artifacts:** `BuildOutput`

7. **Create pipeline**

### **4.2 Repeat for AUTH and CORE Services**

Create similar pipelines for:
- **AUTH:** `ic-auth-staging-pipeline` → `ic-auth-staging-build` → `ic-auth-service-staging`
- **CORE:** `ic-core-staging-pipeline` → `ic-core-staging-build` → `ic-core-service-staging`

## Step 5: Add buildspec.yml to Repositories (15 minutes)

### **Add this file to each microservice repository:**

**File:** `buildspec.yml` (place in repository root)

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

## Step 6: Test Pipeline (15 minutes)

### **6.1 Manual Test**

1. **Go to CodePipeline Console**
2. **Select:** `ic-apigateway-staging-pipeline`
3. **Release change** → Confirm
4. **Monitor progress** through Source → Build → Deploy stages

### **6.2 Verify Deployment**

1. **Go to ECS Console** → Clusters → `ic-general-services-cluster`
2. **Check service:** `ic-api-gateway-service-staging`
3. **Verify:** New task running with updated image
4. **Test ALB:** Confirm application still works

### **6.3 Check Target Group Health**

1. **Go to EC2 Console** → Load Balancers → Target Groups
2. **Select:** `ic-apigateway-staging-tg`
3. **Targets tab:** Verify healthy targets

---

## Day 3 Console Completion Checklist

- ✅ CodePipeline and CodeBuild IAM roles created via Console
- ✅ S3 bucket for pipeline artifacts created
- ✅ CodeBuild projects for all microservices created via Console
- ✅ CodePipeline created and configured via Console
- ✅ buildspec.yml added to GitLab repositories
- ✅ Pipeline tested and working
- ✅ Automated deployments functional

**Final Architecture Achieved:**
```
GitLab → CodePipeline → CodeBuild → ECR → ECS Fargate (Service Connect) → ALB → Users
```

**Result:** Full CI/CD pipeline created using AWS Console with zero-downtime deployments while maintaining Service Connect communication!
