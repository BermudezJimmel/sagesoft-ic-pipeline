# Final Architecture & Implementation Plan

## Corrected Architecture Understanding

### Services in Scope for CI/CD
- **API Gateway:** Port 8000 (Entry point)
- **AUTH API:** Port 8001 (Authentication)
- **CORE:** Port 8002 (Business logic)
- **FILES:** Port 8003 (File management)
- **EMP:** ❌ Not in scope (Already on Amplify)

### Service Communication Flow
```
User → EMP (Amplify) → ALB → API Gateway (8000) → AUTH (8001) → {CORE (8002), FILES (8003)}
```

### Environment Strategy
- **Database:** Same Multi-AZ RDS with different schemas
  - Staging schema: `staging_*`
  - Production schema: `production_*`
- **ECS Services:** Separate services per environment
  - `api-gateway-staging` / `api-gateway-prod`
  - `auth-staging` / `auth-prod`
  - `core-staging` / `core-prod`
  - `files-staging` / `files-prod`

### CI/CD Pipeline Strategy
**GitLab → AWS CodePipeline → CodeBuild → ECR → ECS Fargate**

**Why CodePipeline?**
- GitLab integration built-in (no AWS credentials needed in GitLab)
- Automatic webhook triggers
- Visual pipeline monitoring
- Built-in approval gates

### SSL/Domain
- ✅ Domain available
- ✅ SSL certificate available
- ALB will use HTTPS with existing certificate

### Health Checks Issue
- ❌ No health endpoints currently
- **Solution:** Use TCP health checks on service ports
- **Recommendation:** Add `/health` endpoints later for better monitoring

## Implementation Components

### 1. AWS CodePipeline (Per Service)
- **Source:** GitLab repository
- **Build:** CodeBuild (Docker build + ECR push)
- **Deploy Staging:** ECS update (auto)
- **Approval Gate:** Manual approval for production
- **Deploy Production:** ECS update with circuit breaker

### 2. Load Balancer Configuration
- **Public ALB:** HTTPS with existing SSL certificate
- **Target Group:** API Gateway only
- **Health Check:** TCP on port 8000 (no /health endpoint)

### 3. ECS Service Connect
- **Namespace:** ic-microservices
- **Service Discovery:** auth.local, core.local, files.local
- **No internal ALB needed**

### 4. Database Schema Separation
```sql
-- Staging environment
USE staging_employees;
-- Production environment  
USE production_employees;
```

## Next Steps
1. Create CodePipeline templates
2. Update task definitions with Service Connect
3. Configure ALB with SSL certificate
4. Set up schema-based environment separation
