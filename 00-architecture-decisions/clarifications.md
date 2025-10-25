# Architecture Decisions & Clarifications

## Service Configuration
- **API Gateway App:** Port 8000 (Public via ALB)
- **AUTH API:** Port 8001 (Internal via Service Connect)
- **CORE:** Port 8002 (Internal via Service Connect)
- **FILES:** Port 8003 (Internal via Service Connect)
- **EMP:** Missing port assignment - **NEED CLARIFICATION**

## Database Strategy
- **Single Multi-AZ RDS** for both staging and production
- **Same database instance** shared between environments
- **Secrets Manager:** arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR

## GitLab Integration
- **No AWS credentials configured** - Need to set up
- **GitLab.com** (not on-premises)
- **Separate repositories** for each service

## Pending Clarifications Needed

### 1. EMP Service Port
**Question:** What port does the EMP service use?
**Options:** 8004, or different port?

### 2. Environment Separation Strategy
**Question:** How do you separate staging vs production data in the same RDS?
**Options:** 
- Different database names (staging_db vs production_db)
- Different schemas
- Table prefixes

### 3. GitLab AWS Access
**Question:** How should GitLab access AWS?
**Options:**
- IAM User with Access Keys (simpler)
- OIDC/Federated access (more secure)
- Cross-account roles

### 4. Service Communication Pattern
**Question:** Does API Gateway call all other services, or do services call each other?
**Example:** Does AUTH service call CORE service directly?

### 5. Health Check Endpoints
**Question:** Do all services have /health endpoints?
**Required for:** ALB health checks and ECS health monitoring

### 6. SSL/HTTPS Requirements
**Question:** Do you need HTTPS on the ALB?
**Requires:** SSL certificate (ACM or imported)

### 7. Domain Name
**Question:** What domain will the API Gateway use?
**Example:** api.yourcompany.com or just ALB DNS name?

## Current Architecture Understanding
```
Internet → ALB (HTTPS?) → API Gateway (8000) → Service Connect → {
  AUTH (8001)
  CORE (8002) 
  FILES (8003)
  EMP (????)
} → Multi-AZ RDS
```
