# Day 2: Complete Service Connect + CI/CD (AWS Console Method)

## 🖱️ **Console Implementation Guide**

Complete the remaining services and set up CI/CD pipelines using AWS Console.

---

## Step 1: Update AUTH Service (20 minutes)

### **Update Task Definition:**
1. **Go to:** AWS Console → **ECS** → **Task Definitions**
2. **Find:** AUTH task definition → **Create new revision**
3. **Update Container:**
   - **Port Mappings:** Add name `auth-port`, port 8001
   - **Environment Variables:** Add Service Connect URLs
   - **Secrets:** Add database credentials (same as API Gateway)
4. **Click:** "Create"

### **Update ECS Service:**
5. **Go to:** **ECS** → **Clusters** → Your cluster → **Services**
6. **Find:** AUTH service → **Update service**
7. **Task Definition:** Select latest AUTH revision
8. **Service Connect Configuration:**
   - **Enable Service Connect:** ✅
   - **Namespace:** `ic-microservices`
   - **Port name:** `auth-port`
   - **Discovery name:** `auth-service`
   - **DNS name:** `auth-service.local`
   - **Port:** 8001
9. **Click:** "Update service"

---

## Step 2: Update CORE Service (20 minutes)

### **Repeat Similar Steps:**
1. **Task Definition:** Create new revision with:
   - **Port name:** `core-port`
   - **Container port:** 8002
   - **Environment variables and secrets**

2. **ECS Service:** Update with Service Connect:
   - **Discovery name:** `core-service`
   - **DNS name:** `core-service.local`
   - **Port:** 8002

---

## Step 3: Update FILES Service (20 minutes)

### **Repeat Similar Steps:**
1. **Task Definition:** Create new revision with:
   - **Port name:** `files-port`
   - **Container port:** 8003
   - **Environment variables and secrets**

2. **ECS Service:** Update with Service Connect:
   - **Discovery name:** `files-service`
   - **DNS name:** `files-service.local`
   - **Port:** 8003

---

## Step 4: Create S3 Bucket for CodePipeline (10 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **S3**
2. **Click:** "Create bucket"
3. **Configuration:**
   - **Bucket name:** `ic-microservices-codepipeline-artifacts-RANDOM_SUFFIX`
   - **Region:** ap-southeast-1
   - **Block all public access:** ✅ (keep default)
4. **Click:** "Create bucket"

---

## Step 5: Create CodeBuild Project (25 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **CodeBuild** → **Build projects**
2. **Click:** "Create build project"

### **Project Configuration:**
- **Project name:** `ic-microservices-build-api-gateway`
- **Description:** "Build project for API Gateway microservice"

### **Source:**
- **Source provider:** AWS CodePipeline
- **Build specification:** Use a buildspec file
- **Buildspec name:** `buildspec.yml`

### **Environment:**
- **Environment image:** Managed image
- **Operating system:** Amazon Linux 2
- **Runtime:** Standard
- **Image:** `aws/codebuild/amazonlinux2-x86_64-standard:3.0`
- **Privileged:** ✅ (Enable - required for Docker)

### **Environment Variables:**
Add these variables:
```
IMAGE_REPO_NAME = api-gateway
CONTAINER_NAME = api-gateway
```

### **Service Role:**
- **Service role:** New service role
- **Role name:** `codebuild-ic-microservices-service-role`

3. **Click:** "Create build project"

### **Repeat for Other Services:**
Create 3 more CodeBuild projects:
- `ic-microservices-build-auth` (IMAGE_REPO_NAME=auth, CONTAINER_NAME=auth)
- `ic-microservices-build-core` (IMAGE_REPO_NAME=core, CONTAINER_NAME=core)  
- `ic-microservices-build-files` (IMAGE_REPO_NAME=files, CONTAINER_NAME=files)

---

## Step 6: Create CodePipeline (30 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **CodePipeline** → **Pipelines**
2. **Click:** "Create pipeline"

### **Pipeline Settings:**
- **Pipeline name:** `ic-microservices-api-gateway-pipeline`
- **Service role:** New service role
- **Artifact store:** Custom location
- **Bucket:** Select S3 bucket created in Step 4

### **Source Stage:**
3. **Source provider:** GitLab
4. **Connection:** Create new connection (follow OAuth setup)
5. **Repository:** `CLIENT_TO_REPLACE/api-gateway-repo`
6. **Branch:** `main`
7. **Output artifacts:** `SourceOutput`

### **Build Stage:**
8. **Build provider:** AWS CodeBuild
9. **Project name:** `ic-microservices-build-api-gateway`
10. **Input artifacts:** `SourceOutput`
11. **Output artifacts:** `BuildOutput`

### **Deploy Staging Stage:**
12. **Deploy provider:** Amazon ECS
13. **Cluster name:** Your ECS cluster name
14. **Service name:** `api-gateway-staging`
15. **Image definitions file:** `imagedefinitions.json`
16. **Input artifacts:** `BuildOutput`

### **Approval Stage:**
17. **Action provider:** Manual approval
18. **Action name:** `ManualApproval`
19. **Review URL:** (optional)
20. **Comments:** "Please review staging deployment and approve for production"

### **Deploy Production Stage:**
21. **Deploy provider:** Amazon ECS
22. **Cluster name:** Your ECS cluster name
23. **Service name:** `api-gateway-prod`
24. **Image definitions file:** `imagedefinitions.json`
25. **Input artifacts:** `BuildOutput`

26. **Click:** "Create pipeline"

---

## Step 7: Setup GitLab Connection (15 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **CodePipeline** → **Settings** → **Connections**
2. **Click:** "Create connection"
3. **Provider:** GitLab
4. **Connection name:** `gitlab-connection`
5. **Click:** "Connect to GitLab"
6. **Follow OAuth flow:** Authorize AWS access to GitLab
7. **Test connection:** Verify it shows "Available"

### **Update Pipeline Source:**
8. **Go back to your pipeline** → **Edit**
9. **Edit Source stage** → Select the new GitLab connection
10. **Save changes**

---

## Step 8: Test Complete Setup (20 minutes)

### **Test Service Connect:**
1. **Go to:** AWS Console → **ECS** → **Clusters** → Your cluster
2. **Click:** Any service → **Tasks** tab → Click task ID
3. **Click:** "Execute command" (if available) or check logs
4. **Verify:** Services can resolve `.local` DNS names

### **Test ALB:**
5. **Browser test:** `https://YOUR_ALB_DNS_NAME`
6. **Expected:** API Gateway responds through ALB

### **Test CodePipeline:**
7. **Go to:** **CodePipeline** → Your pipeline
8. **Click:** "Release change" to trigger manually
9. **Monitor:** Each stage should complete successfully
10. **Test approval:** Approve manual approval stage
11. **Verify:** Production deployment completes

---

## ✅ **Day 2 Console Completion Checklist**
- [ ] AUTH service updated with Service Connect (`auth-service.local:8001`)
- [ ] CORE service updated with Service Connect (`core-service.local:8002`)
- [ ] FILES service updated with Service Connect (`files-service.local:8003`)
- [ ] S3 bucket created for CodePipeline artifacts
- [ ] CodeBuild projects created for all 4 services
- [ ] CodePipeline created with GitLab source
- [ ] GitLab connection established and working
- [ ] Manual approval gate configured
- [ ] End-to-end pipeline test successful
- [ ] All services communicate via Service Connect
- [ ] ALB serves API Gateway over HTTPS

---

## 🔄 **Alternative: Use CLI Method**
If you prefer command-line approach, use: [Day 2 CLI Guide](./day2-complete-setup.md)

---

## 🎉 **Implementation Complete!**

### **Final Architecture Achieved:**
```
EMP (Amplify) → ALB (HTTPS) → API Gateway (8000) → Service Connect → {AUTH (8001), CORE (8002), FILES (8003)} → Multi-AZ RDS
                                                                                                                    ↑
GitLab Push → CodePipeline → CodeBuild → ECR → ECS Staging → Manual Approval → ECS Production ──────────────────┘
```

**Your microservices now have:**
- ✅ **Secure load balancer** with SSL
- ✅ **Internal service discovery** via Service Connect  
- ✅ **Automated CI/CD** with manual approval gates
- ✅ **Zero downtime deployments** with rollback capability

**Next Step:** Client must update application code - see [Code Migration Guide](../07-application-code-updates/code-migration-guide.md)
