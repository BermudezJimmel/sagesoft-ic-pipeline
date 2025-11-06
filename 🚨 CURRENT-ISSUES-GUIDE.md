# 🚨 Current Implementation Issues & Solutions

## 🔥 **Issue 1: CodeDeploy Internal ALB Error**

### **Error:**
```
Primary taskset target group must be behind listener arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:listener/app/ic-corev3-production-internal-lb/728af0d283c767fa/c83337b653f9a017
```

### **Root Cause:**
Internal ALB listener doesn't have both blue and green target groups configured for CodeDeploy Blue/Green deployment.

### **Solution:**
```bash
# Fix internal ALB listener with both target groups
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:listener/app/ic-corev3-production-internal-lb/728af0d283c767fa/c83337b653f9a017 \
  --default-actions Type=forward,ForwardConfig='{
    "TargetGroups": [
      {
        "TargetGroupArn": "arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/ic-corev3-production-blue-tg/e00a29ace65bd46b",
        "Weight": 100
      },
      {
        "TargetGroupArn": "arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/ic-corev3-production-green-tg/024d6f974ed7d92a",
        "Weight": 0
      }
    ]
  }' \
  --region ap-southeast-1
```

---

## 🔥 **Issue 2: API Gateway → CORE ALB 504 Error**

### **Problem:**
API Gateway (private subnet) → CORE ALB (public subnet) fails with VPC CIDR security group rule but works with 0.0.0.0/0.

### **Root Cause:**
Traffic goes through NAT Gateway, so source IP becomes NAT Gateway's public IP (not VPC CIDR).

### **Solutions:**

#### **Option A: Use NAT Gateway IP (Secure)**
```bash
# Find NAT Gateway IP
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=vpc-0a4ae3fd58a788210" \
  --query 'NatGateways[*].NatGatewayAddresses[*].PublicIp' \
  --region ap-southeast-1

# Allow NAT Gateway IP in CORE ALB security group
aws ec2 authorize-security-group-ingress \
  --group-id YOUR_CORE_ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr NAT_GATEWAY_IP/32 \
  --region ap-southeast-1
```

#### **Option B: Internal ALB (Most Secure)**
```bash
# Create internal ALB (already done)
# Traffic stays within VPC, VPC CIDR rules work
```

---

## 🔥 **Issue 3: Service Connect vs ALB Confusion**

### **Understanding:**
- **Service Connect:** DNS resolution only (`core-service.local` → IP list)
- **ALB:** Actual load balancing with health checks

### **With Multiple Tasks:**
```bash
# Service Connect returns: [10.0.1.100, 10.0.1.101]
# Client picks first IP (usually same task gets all traffic)
# No health checks, no automatic failover
```

### **Solution:**
Use **both** for different purposes:
- **Service Connect:** Internal service discovery
- **Internal ALB:** Load balancing and Blue/Green deployments

---

## 🔥 **Issue 4: Internet-facing vs Internal ALB**

### **Current Status:**
- CORE ALB: `ic-corev3-production-ecs-lb` (internet-facing)
- Need: Internal ALB for secure VPC-only communication

### **Cannot Modify Existing ALB:**
AWS doesn't allow changing ALB scheme from internet-facing to internal.

### **Solution:**
Create new internal ALB and migrate:
```bash
# 1. Create internal ALB (done)
# 2. Configure listener with blue/green target groups (in progress)
# 3. Update CodeDeploy deployment group
# 4. Test API Gateway → Internal ALB
# 5. Delete old internet-facing ALB
```

---

## 🎯 **Current Priority Order:**

1. **Fix Internal ALB Listener** (CodeDeploy error)
2. **Test Internal ALB Communication** (API Gateway → CORE)
3. **Update CodeDeploy Deployment Group** (use internal ALB)
4. **Proceed with CI/CD Pipeline Setup**

---

## 📋 **Verification Commands:**

### **Check Internal ALB Listener:**
```bash
aws elbv2 describe-listeners \
  --listener-arns arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:listener/app/ic-corev3-production-internal-lb/728af0d283c767fa/c83337b653f9a017 \
  --region ap-southeast-1
```

### **Test Internal ALB:**
```bash
# From API Gateway container (if execute-command enabled)
curl -v http://INTERNAL_ALB_DNS_NAME
```

### **Check Target Health:**
```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-1:795189341938:targetgroup/ic-corev3-production-blue-tg/e00a29ace65bd46b \
  --region ap-southeast-1
```

---

**🚀 Once these issues are resolved, we can proceed with Day 3 CI/CD pipeline setup!**
