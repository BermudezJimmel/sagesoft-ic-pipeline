# 🗄️ Database Updates for Service Connect

## 🎯 **Understanding the Architecture**

### **Client's Setup:**
- ✅ **API Gateway stores microservice URLs in database** (not environment variables)
- ✅ **API Gateway reads URLs from database at runtime**
- ✅ **We need to update the database with Service Connect URLs**
- ✅ **Keep existing secrets** (like FILES_MICROSERVICE_URL with random string)

### **Why Database Updates Are Needed:**
```
Before Service Connect: API Gateway → Database → "http://54.123.45.67:8001" (public IP)
After Service Connect:  API Gateway → Database → "http://auth-service.local:8001" (Service Connect)
```

---

## 📋 **Database Update Required**

After Service Connect is implemented, you **MUST** update the database with Service Connect URLs.

### **Example SQL Updates:**
```sql
-- Update AUTH service URL
UPDATE microservices_config SET 
  url = 'http://auth-service.local:8001' 
WHERE service_name = 'auth';

-- Update CORE service URL
UPDATE microservices_config SET 
  url = 'http://core-service.local:8002' 
WHERE service_name = 'core';

-- Update FILES service URL
UPDATE microservices_config SET 
  url = 'http://files-service.local:8003' 
WHERE service_name = 'files';
```

---

## ❓ **Client Information Needed**

### **Database Structure Questions:**
1. **Table name:** What table stores the microservice URLs?
2. **Column names:** What are the exact column names?
   - Service identifier column: `service_name`? `service_id`? `microservice_name`?
   - URL column: `url`? `endpoint`? `service_url`?
3. **Service identifiers:** How are services identified in the database?
   - `'auth'`, `'core'`, `'files'`?
   - Or `'AUTH_SERVICE'`, `'CORE_SERVICE'`, `'FILES_SERVICE'`?

### **Example Database Schema:**
```sql
-- Possible table structure (client to confirm)
CREATE TABLE microservices_config (
    id INT PRIMARY KEY,
    service_name VARCHAR(50),    -- 'auth', 'core', 'files'
    url VARCHAR(255),            -- Service URL
    status ENUM('active', 'inactive'),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

---

## 🕐 **When to Update Database**

### **Option 1: Before Service Connect (Recommended)**
```
1. Update database with Service Connect URLs
2. Deploy Service Connect
3. Test connectivity
```

### **Option 2: After Service Connect**
```
1. Deploy Service Connect
2. Update database with Service Connect URLs
3. Test connectivity
```

**Recommendation:** Update database **before** Service Connect deployment to avoid downtime.

---

## 🔧 **Implementation Steps**

### **Step 1: Get Database Information**
Client provides:
- Table name
- Column names  
- Current service identifiers
- Sample current data

### **Step 2: Create Migration Script**
```sql
-- Template migration script (client to customize)
-- Backup current URLs first
CREATE TABLE microservices_config_backup AS 
SELECT * FROM microservices_config;

-- Update to Service Connect URLs
UPDATE microservices_config SET 
  url = 'http://auth-service.local:8001',
  updated_at = NOW()
WHERE service_name = 'AUTH_SERVICE_IDENTIFIER';

UPDATE microservices_config SET 
  url = 'http://core-service.local:8002',
  updated_at = NOW()
WHERE service_name = 'CORE_SERVICE_IDENTIFIER';

UPDATE microservices_config SET 
  url = 'http://files-service.local:8003',
  updated_at = NOW()
WHERE service_name = 'FILES_SERVICE_IDENTIFIER';
```

### **Step 3: Test Migration**
```sql
-- Verify updates
SELECT service_name, url FROM microservices_config 
WHERE service_name IN ('AUTH_IDENTIFIER', 'CORE_IDENTIFIER', 'FILES_IDENTIFIER');
```

### **Step 4: Rollback Plan**
```sql
-- If needed, rollback from backup
DELETE FROM microservices_config;
INSERT INTO microservices_config SELECT * FROM microservices_config_backup;
```

---

## ✅ **Corrected Task Definition**

**No environment variables needed** - API Gateway reads URLs from database:

```json
{
  "family": "ic-apigateway-staging-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-execution-role",
  "taskRoleArn": "arn:aws:iam::795189341938:role/ic-apigateway-staging-task-role",
  "containerDefinitions": [
    {
      "name": "ic-api-gateway-container",
      "image": "795189341938.dkr.ecr.ap-southeast-1.amazonaws.com/ic-api-gateway-image:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "name": "ic-api-gateway-port",
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        "... all existing secrets remain the same ..."
      ],
      "environment": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ic-api-gateway-staging-logs",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
```

---

## 🚨 **Critical Success Factors**

1. **Database backup** before any changes
2. **Test in staging** before production
3. **Coordinate timing** with Service Connect deployment
4. **Verify connectivity** after updates
5. **Have rollback plan** ready

---

## 📋 **Client Action Items**

- [ ] Provide database table structure
- [ ] Provide current service identifiers
- [ ] Review and approve migration script
- [ ] Schedule database update timing
- [ ] Test database connectivity after updates

**This database update is CRITICAL for Service Connect to work with the client's architecture!** 🎯
