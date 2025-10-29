# Client Configuration Checklist

## Before Implementation - Client Must Provide

### 1. AWS Infrastructure Details
- [ ] **VPC ID:** `vpc-xxxxxxxxx`
- [ ] **Public Subnets:** `subnet-xxxxxxxx`, `subnet-yyyyyyyy` (for ALB)
- [ ] **Private Subnets:** `subnet-aaaaaaaa`, `subnet-bbbbbbbb` (for ECS)
- [ ] **ECS Cluster Name:** Current cluster name
- [ ] **Security Group IDs:** ALB security group, ECS security group

### 2. Current ECS Service Names
- [ ] **API Gateway Service:** Current service name
- [ ] **AUTH Service:** Current service name  
- [ ] **CORE Service:** Current service name
- [ ] **FILES Service:** Current service name

### 3. GitLab Repository Information
- [ ] **API Gateway Repo:** `https://gitlab.com/your-org/api-gateway-repo`
- [ ] **AUTH Repo:** `https://gitlab.com/your-org/auth-repo`
- [ ] **CORE Repo:** `https://gitlab.com/your-org/core-repo`
- [ ] **FILES Repo:** `https://gitlab.com/your-org/files-repo`

### 4. Database Schema Configuration
- [ ] **Staging Schema Name:** `staging_employees` (or client preference)
- [ ] **Production Schema Name:** `production_employees` (or client preference)

### 5. IAM Roles (Create if not exist)
- [ ] **CodePipeline Service Role:** `CodePipelineServiceRole`
- [ ] **CodeBuild Service Role:** `CodeBuildServiceRole`
- [ ] **ECS Task Execution Role:** `ecsTaskExecutionRole`
- [ ] **ECS Task Role:** `ecsTaskRole`

## During Implementation - Replace Placeholders

### In All Configuration Files, Replace:
```bash
# Infrastructure
REPLACE_WITH_YOUR_VPC_ID → vpc-actual-id
REPLACE_PUBLIC_SUBNET_1 → subnet-actual-public-1
REPLACE_PUBLIC_SUBNET_2 → subnet-actual-public-2
REPLACE_WITH_ALB_SECURITY_GROUP → sg-actual-alb-sg
REPLACE_WITH_YOUR_CLUSTER_NAME → actual-cluster-name

# Services
REPLACE_WITH_API_GATEWAY_SERVICE_NAME → actual-api-gateway-service
REPLACE_WITH_AUTH_SERVICE_NAME → actual-auth-service
REPLACE_WITH_CORE_SERVICE_NAME → actual-core-service
REPLACE_WITH_FILES_SERVICE_NAME → actual-files-service

# Database
REPLACE_WITH_SCHEMA_NAME → staging_employees (for staging)
REPLACE_WITH_SCHEMA_NAME → production_employees (for production)

# Generated Values (from AWS commands)
REPLACE_WITH_NAMESPACE_ID_FROM_STEP1 → ns-generated-id
REPLACE_WITH_ALB_ID → generated-alb-id
REPLACE_WITH_TG_ID → generated-target-group-id
RANDOM_SUFFIX → unique-suffix-for-s3-bucket
```

## Post-Implementation - Client Configuration

### 1. Database Updates (CRITICAL - REQUIRED)
**Client MUST update microservice URLs in database - see:** [Database Updates Guide](../09-database-updates/service-connect-database-migration.md)

**Client Must Provide:**
- Database table name where URLs are stored
- Column names (service identifier, URL column)
- Current service identifiers in database
- Sample current data structure

**Database Update Example:**
```sql
UPDATE microservices_config SET 
  url = 'http://auth-service.local:8001' 
WHERE service_name = 'auth';
```

### 2. Application Code Updates (Optional)
**Only needed for non-API Gateway services - see:** [Application Code Updates Guide](../07-application-code-updates/code-migration-guide.md)

**Note:** API Gateway reads URLs from database, so no code changes needed for API Gateway.

### 2. GitLab Repository Setup
Each repository needs `buildspec.yml` in root:
```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 795189341938.dkr.ecr.ap-southeast-1.amazonaws.com
      - REPOSITORY_URI=795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - docker build -t $IMAGE_REPO_NAME:latest .
      - docker tag $IMAGE_REPO_NAME:latest $REPOSITORY_URI:latest
      - docker tag $IMAGE_REPO_NAME:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
```

### 2. CodePipeline Source Configuration
In AWS Console → CodePipeline → Source Stage:
- **Provider:** GitLab
- **Server URL:** `https://gitlab.com`
- **Repository:** `your-org/service-name`
- **Branch:** `main`
- **Authentication:** OAuth (follow AWS setup wizard)

### 3. Application Code Updates
Update service URLs in application code:
```javascript
// Before (hardcoded IPs)
const authUrl = "http://54.123.45.67:8001";

// After (Service Connect)
const authUrl = "http://auth-service.local:8001";
```

### 4. Database Connection Updates
Update database connection to use schemas:
```javascript
// Staging environment
const dbConfig = {
  host: process.env.DB_HOST,
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_SCHEMA // "staging_employees"
};
```

## Success Verification
- [ ] ALB responds via HTTPS
- [ ] API Gateway accessible through ALB
- [ ] Services communicate via Service Connect
- [ ] CodePipeline triggers on GitLab push
- [ ] Staging deployment works automatically
- [ ] Manual approval gate functions
- [ ] Production deployment works after approval
- [ ] Rollback capability tested

## Support Contacts
- **AWS Support:** For infrastructure issues
- **Implementation Team:** For configuration questions
- **Documentation:** All guides in `/CICD-LoadBalancer-Implementation/` folder
