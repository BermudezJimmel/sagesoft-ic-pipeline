# Day 1: ALB + Service Connect (AWS Console Method)

## 🖱️ **Console Implementation Guide**

Choose this method if you prefer clicking through the AWS web interface instead of using command line.

---

## Step 0: Create IAM Roles (20 minutes) - **REQUIRED FIRST**

**⚠️ IMPORTANT:** Create IAM roles before task definitions!

### **Console Steps:**
1. **Go to:** AWS Console → **IAM** → **Roles**
2. **Click:** "Create role"
3. **Select:** "AWS service" → "Elastic Container Service" → "Elastic Container Service Task"
4. **Role name:** `ic-apigateway-staging-execution-role`
5. **Attach policies:**
   - `AmazonECSTaskExecutionRolePolicy`
   - `SecretsManagerReadWrite`
6. **Repeat for task role:** `ic-apigateway-staging-task-role`

**For complete steps:** [IAM Roles Creation Guide](../08-iam-roles-setup/iam-roles-creation.md)

---

## Step 1: Create Service Discovery Namespace (10 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **Cloud Map** service
2. **Click:** "Create namespace"
3. **Configure:**
   - **Namespace type:** Private DNS namespace
   - **Namespace name:** `ic-microservices`
   - **VPC:** Select your VPC (where ECS services run)
   - **Description:** "Service discovery for IC microservices"
4. **Click:** "Create namespace"
5. **Save the Namespace ID** (you'll need it later)

---

## Step 2: Create Application Load Balancer (20 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **EC2** → **Load Balancers**
2. **Click:** "Create Load Balancer"
3. **Select:** "Application Load Balancer"

### **Basic Configuration:**
- **Name:** `ic-microservices-alb`
- **Scheme:** Internet-facing
- **IP address type:** IPv4

### **Network Mapping:**
- **VPC:** Select your VPC
- **Availability Zones:** Select 2+ public subnets
- **Security Groups:** Select ALB security group (allow 80, 443)

### **Listeners:**
- **Protocol:** HTTPS
- **Port:** 443
- **SSL Certificate:** Select existing certificate
  - **ARN:** `arn:aws:acm:ap-southeast-1:795189341938:certificate/7cd7d7d8-f9ad-40b1-a0ca-e56472345fa4`

4. **Click:** "Create load balancer"

---

## Step 3: Create Target Group (15 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **EC2** → **Target Groups**
2. **Click:** "Create target group"

### **Configuration:**
- **Target type:** IP addresses
- **Target group name:** `api-gateway-tg`
- **Protocol:** HTTP
- **Port:** 8000
- **VPC:** Select your VPC

### **Health Check Settings:**
- **Health check protocol:** TCP
- **Health check port:** 8000
- **Health check interval:** 30 seconds
- **Healthy threshold:** 2
- **Unhealthy threshold:** 3

3. **Click:** "Create target group"
4. **Go back to ALB** → **Listeners** → **Edit HTTPS:443 listener**
5. **Set Default Action:** Forward to `api-gateway-tg`

---

## Step 4: Update API Gateway Task Definition (25 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **ECS** → **Task Definitions**
2. **Find:** Your current API Gateway task definition
3. **Click:** Task definition name → **Create new revision**

### **Container Configuration:**
4. **Click:** Container name to edit
5. **Update Port Mappings:**
   - **Name:** `api-port`
   - **Container port:** 8000
   - **Protocol:** TCP

### **Environment Variables:**
6. **Add Environment Variables:**
   ```
   DB_SCHEMA = REPLACE_WITH_SCHEMA_NAME
   AUTH_SERVICE_URL = http://auth-service.local:8001
   CORE_SERVICE_URL = http://core-service.local:8002
   FILES_SERVICE_URL = http://files-service.local:8003
   ```

### **Secrets (from Secrets Manager):**
7. **Add Secrets:**
   - **DB_HOST:** `arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:host::`
   - **DB_USERNAME:** `arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:username::`
   - **DB_PASSWORD:** `arn:aws:secretsmanager:ap-southeast-1:795189341938:secret:ic-microservices-rds-a7igAR:password::`

8. **Click:** "Create" to save new task definition revision

---

## Step 5: Update API Gateway ECS Service (20 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **ECS** → **Clusters** → Your cluster
2. **Click:** **Services** tab
3. **Find:** API Gateway service → **Update service**

### **Service Configuration:**
4. **Task Definition:**
   - **Family:** Select updated API Gateway task definition
   - **Revision:** Select latest (LATEST)

### **Load Balancer Configuration:**
5. **Load Balancer Type:** Application Load Balancer
6. **Load Balancer:** Select `ic-microservices-alb`
7. **Container to Load Balance:**
   - **Container name:port:** `api-gateway:8000`
   - **Target group:** `api-gateway-tg`

### **Service Connect Configuration:**
8. **Turn on Service Connect:** ✅ Enabled
9. **Namespace:** Select `ic-microservices` (created in Step 1)
10. **Port Mapping Configuration:**
    - **Port name:** `api-port`
    - **Discovery name:** `api-gateway`
    - **DNS name:** `api-gateway.local`
    - **Port:** 8000

11. **Click:** "Update service"

---

## Step 6: Test ALB Connection (10 minutes)

### **AWS Console Steps:**
1. **Go to:** AWS Console → **EC2** → **Load Balancers**
2. **Find:** `ic-microservices-alb`
3. **Copy DNS Name:** (e.g., `ic-microservices-alb-123456789.ap-southeast-1.elb.amazonaws.com`)

### **Test Connection:**
4. **Open browser:** `https://YOUR_ALB_DNS_NAME`
5. **Expected:** Response from API Gateway service
6. **Check Target Group Health:**
   - Go to **Target Groups** → `api-gateway-tg` → **Targets** tab
   - Status should show "Healthy"

---

## ✅ **Day 1 Console Completion Checklist**
- [ ] Service Discovery namespace `ic-microservices` created
- [ ] ALB `ic-microservices-alb` created with SSL certificate
- [ ] Target group `api-gateway-tg` created with TCP health checks
- [ ] API Gateway task definition updated with Service Connect ports
- [ ] API Gateway service updated with Service Connect and ALB
- [ ] HTTPS connection working through ALB
- [ ] Target group shows "Healthy" status

---

## 🔄 **Alternative: Use CLI Method**
If you prefer command-line approach, use: [Day 1 CLI Guide](./day1-alb-service-connect.md)

**Next:** [Day 2 Console Guide](./day2-console-guide.md) - Complete Service Connect + CI/CD setup
