# ⚠️ ALB Scheme Selection Guide - Critical Decision

## 🚨 **COMMON MISTAKE: Wrong ALB Scheme Selection**

### **The Problem:**
Many developers accidentally select **"Internet-facing"** when they need **"Internal"** ALB, causing:
- 504 errors for internal communication
- Security group issues (NAT Gateway IP vs VPC CIDR)
- Unnecessary internet exposure

---

## 🎯 **When to Use Each ALB Type:**

### **Internet-facing ALB**
```
Internet → Internet-facing ALB → ECS Services
```
**Use for:**
- External user access (web applications)
- API endpoints called from internet
- Public-facing services

**Example:** API Gateway ALB for external Amplify frontend

### **Internal ALB**
```
VPC Services → Internal ALB → ECS Services
```
**Use for:**
- Service-to-service communication
- Internal microservices
- Database access services
- Backend services not exposed to internet

**Example:** CORE service ALB for API Gateway → CORE communication

---

## 🔧 **How to Choose Correctly:**

### **Ask These Questions:**
1. **Who calls this service?**
   - Internet users → **Internet-facing**
   - Other AWS services in VPC → **Internal**

2. **Where are the callers located?**
   - Outside VPC → **Internet-facing**
   - Inside VPC → **Internal**

3. **Security requirement?**
   - Public access needed → **Internet-facing**
   - VPC-only access → **Internal**

---

## 🏗️ **Your IC Microservices Architecture:**

### **Correct Setup:**
```
Internet → API Gateway ALB (Internet-facing) → API Gateway Service
                    ↓
API Gateway → CORE ALB (Internal) → CORE Service
API Gateway → AUTH ALB (Internal) → AUTH Service  
API Gateway → FILES ALB (Internal) → FILES Service
```

### **ALB Scheme Decisions:**
- **API Gateway ALB:** Internet-facing (Amplify frontend calls it)
- **CORE ALB:** Internal (only API Gateway calls it)
- **AUTH ALB:** Internal (only API Gateway calls it)
- **FILES ALB:** Internal (only API Gateway calls it)

---

## 🛠️ **Console Steps (Correct Way):**

### **For Internal ALB:**
1. EC2 → Load Balancers → Create Application Load Balancer
2. **Name:** `ic-core-internal-alb`
3. **⚠️ CRITICAL:** Scheme = **"Internal"** (not Internet-facing)
4. **VPC:** Select your VPC
5. **Subnets:** Select **private subnets** (same as ECS services)
6. **Security Groups:** Create/select SG allowing VPC CIDR (10.0.0.0/16)

### **For Internet-facing ALB:**
1. EC2 → Load Balancers → Create Application Load Balancer
2. **Name:** `ic-api-gateway-alb`
3. **Scheme:** "Internet-facing"
4. **VPC:** Select your VPC
5. **Subnets:** Select **public subnets**
6. **Security Groups:** Create/select SG allowing internet (0.0.0.0/0)

---

## 🔍 **Verification Commands:**

### **Check ALB Scheme:**
```bash
# Verify ALB is internal
aws elbv2 describe-load-balancers \
  --names ic-core-internal-alb \
  --query 'LoadBalancers[0].{Name:LoadBalancerName,Scheme:Scheme,Subnets:AvailabilityZones[*].SubnetId}' \
  --region ap-southeast-1

# Should show:
# "Scheme": "internal"
# "Subnets": ["subnet-096a5e3c10eef5f5c", "subnet-0e9a0ec15dc80197d"] (private)
```

### **Test Internal Communication:**
```bash
# From API Gateway, should work with VPC CIDR security group
curl -v http://INTERNAL_ALB_DNS_NAME

# No NAT Gateway IP issues because traffic stays in VPC
```

---

## 🚨 **Red Flags (Wrong Setup):**

### **Signs You Selected Wrong Scheme:**
- ✅ ALB created successfully
- ❌ 504 errors when calling from other VPC services
- ❌ Security group VPC CIDR rules don't work
- ❌ Need to allow 0.0.0.0/0 for internal communication
- ❌ NAT Gateway IP complications

### **If You Made This Mistake:**
1. **Cannot modify existing ALB scheme** (AWS limitation)
2. **Must create new ALB** with correct scheme
3. **Update all references** to new ALB DNS name
4. **Delete old ALB** after testing

---

## 💡 **Pro Tips:**

### **Naming Convention:**
- `service-name-external-alb` (Internet-facing)
- `service-name-internal-alb` (Internal)

### **Security Groups:**
- **Internet-facing ALB:** Allow 0.0.0.0/0 on port 80/443
- **Internal ALB:** Allow VPC CIDR (10.0.0.0/16) on required ports

### **Subnet Selection:**
- **Internet-facing ALB:** Public subnets (with Internet Gateway route)
- **Internal ALB:** Private subnets (same as ECS services)

---

**🎯 Remember: This single checkbox selection determines your entire network architecture and security model!**
