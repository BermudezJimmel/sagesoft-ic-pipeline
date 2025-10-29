# 🔧 Application Code Updates Guide

## 📋 **What Client Must Update**

### **🎯 Client's Architecture Understanding:**
- ✅ **API Gateway stores microservice URLs in database** (not environment variables)
- ✅ **API Gateway reads URLs from database at runtime**
- ✅ **Database must be updated with Service Connect URLs**
- ✅ **No code changes needed** - only database updates

---

## 🗄️ **Database Updates Required (CRITICAL)**

**Primary Task:** Update database with Service Connect URLs

See complete guide: [Database Updates for Service Connect](../09-database-updates/service-connect-database-migration.md)

### **Quick Summary:**
```sql
-- Example updates (client to customize)
UPDATE microservices_config SET 
  url = 'http://auth-service.local:8001' 
WHERE service_name = 'auth';

UPDATE microservices_config SET 
  url = 'http://core-service.local:8002' 
WHERE service_name = 'core';

UPDATE microservices_config SET 
  url = 'http://files-service.local:8003' 
WHERE service_name = 'files';
```

---

## 🔍 **Optional: Code Updates for Other Services**

If AUTH, CORE, or FILES services need to call each other directly:

### **Environment Variables (For Non-API Gateway Services)**
```bash
# In AUTH service .env or config
CORE_SERVICE_URL=http://core-service.local:8002
FILES_SERVICE_URL=http://files-service.local:8003

# In CORE service .env or config  
AUTH_SERVICE_URL=http://auth-service.local:8001
FILES_SERVICE_URL=http://files-service.local:8003
```

### **Configuration Files**
```json
// config.json for microservices
{
  "services": {
    "auth": "http://auth-service.local:8001",
    "core": "http://core-service.local:8002", 
    "files": "http://files-service.local:8003"
  }
}
```

---

## 🎯 **Step 3: Service-Specific Updates**

### **API Gateway Service Updates**
```javascript
// API Gateway calls other services
const authResponse = await fetch('http://auth-service.local:8001/login', {
  method: 'POST',
  body: JSON.stringify(credentials)
});

const userData = await fetch('http://core-service.local:8002/users/' + userId);
const fileData = await fetch('http://files-service.local:8003/upload');
```

### **AUTH Service Updates** 
```javascript
// AUTH service might call CORE for user validation
const userProfile = await fetch('http://core-service.local:8002/profile/' + userId);
```

### **CORE Service Updates**
```javascript
// CORE service might call FILES for document retrieval
const documents = await fetch('http://files-service.local:8003/documents/' + userId);
```

---

## ⚠️ **Important Notes**

### **Service Connect URLs Only Work:**
- ✅ **Inside ECS containers** with Service Connect enabled
- ❌ **NOT from your laptop** or external systems
- ❌ **NOT from other AWS services** (Lambda, EC2 without Service Connect)

### **External Access Still Uses ALB:**
```javascript
// External clients (EMP Amplify) still use ALB
const apiUrl = "https://ic-microservices-alb-123456789.ap-southeast-1.elb.amazonaws.com";

// Internal service-to-service uses Service Connect
const authUrl = "http://auth-service.local:8001";
```

---

## 📋 **Client Checklist**

### **Before Implementation:**
- [ ] Identify all hardcoded service IPs in code
- [ ] List all configuration files that need updates
- [ ] Backup current code before changes

### **During Implementation:**
- [ ] Update environment variables in code
- [ ] Update configuration files
- [ ] Test locally with environment variables
- [ ] Commit changes to GitLab repositories

### **After Implementation:**
- [ ] Verify services can communicate via Service Connect
- [ ] Test all API endpoints work through ALB
- [ ] Monitor logs for any connection errors

---

## 🔧 **Testing Service Connect Communication**

### **From inside API Gateway container:**
```bash
# Test if Service Connect DNS resolution works
nslookup auth-service.local
nslookup core-service.local
nslookup files-service.local

# Test HTTP connectivity
curl http://auth-service.local:8001/health
curl http://core-service.local:8002/health
curl http://files-service.local:8003/health
```

---

## 🚨 **Troubleshooting**

| Issue | Cause | Solution |
|-------|-------|----------|
| `auth-service.local` not found | Service Connect not enabled | Check ECS service configuration |
| Connection refused | Wrong port number | Verify port mappings in task definition |
| Timeout errors | Service not healthy | Check ECS service health in console |
| Works locally, fails in ECS | Using old hardcoded IPs | Update all service URLs in code |

---

## ✅ **Success Verification**

After code updates, you should see:
- ✅ API Gateway can call AUTH service via `auth-service.local:8001`
- ✅ AUTH service can call CORE service via `core-service.local:8002`
- ✅ All services can call FILES service via `files-service.local:8003`
- ✅ External clients can reach API Gateway via ALB HTTPS URL
- ✅ No hardcoded IP addresses remain in code

**This completes the Service Connect migration! 🎉**
