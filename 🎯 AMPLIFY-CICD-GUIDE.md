# 🎯 AWS Amplify CI/CD Implementation Guide for EMP UI

## 📊 **Overview**

**Goal:** Set up automated CI/CD pipeline for EMP UI frontend using AWS Amplify's built-in capabilities  
**Source:** GitLab repository  
**Target:** Amplify hosting with automatic deployments  
**Timeline:** 30-60 minutes setup  

---

## 🚀 **Step 1: Connect GitLab Repository (10 minutes)**

### **Via AWS Console:**
1. **Navigate to Amplify Console**
   - Go to AWS Amplify service
   - Click "Get Started" under "Amplify Hosting"

2. **Connect Repository**
   - Select "GitLab" as source
   - Click "Continue"
   - Authorize AWS Amplify to access your GitLab account
   - Select your EMP UI repository
   - Choose branch (usually `main` or `master`)

### **Via AWS CLI:**
```bash
# Create Amplify app connected to GitLab
aws amplify create-app \
  --name "emp-ui-app" \
  --repository "https://gitlab.com/YOUR_ORG/emp-ui" \
  --platform WEB \
  --iam-service-role-arn "arn:aws:iam::795189341938:role/amplifyconsole-backend-role" \
  --region ap-southeast-1
```

---

## ⚙️ **Step 2: Configure Build Settings (15 minutes)**

### **Auto-Detected Build Settings:**
Amplify will automatically detect your frontend framework and suggest build settings.

### **Custom Build Configuration (amplify.yml):**
Create `amplify.yml` in your repository root:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist  # or build/ for React
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

### **Environment Variables:**
```bash
# Set environment variables via console or CLI
aws amplify put-backend-environment \
  --app-id YOUR_APP_ID \
  --environment-name production \
  --deployment-artifacts '{
    "API_GATEWAY_URL": "https://your-api-gateway-url.com",
    "NODE_ENV": "production"
  }' \
  --region ap-southeast-1
```

---

## 🌿 **Step 3: Configure Branch Deployment (10 minutes)**

### **Production Branch (main):**
```bash
# Create production branch deployment
aws amplify create-branch \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --framework "React" \
  --enable-auto-build \
  --region ap-southeast-1
```

### **Staging Branch (develop):**
```bash
# Create staging branch deployment
aws amplify create-branch \
  --app-id YOUR_APP_ID \
  --branch-name develop \
  --framework "React" \
  --enable-auto-build \
  --region ap-southeast-1
```

### **Branch Settings:**
- **Production (main):** Auto-deploy enabled
- **Staging (develop):** Auto-deploy enabled with preview URL
- **Feature branches:** Manual deploy or auto-preview

---

## 🔧 **Step 4: Advanced Configuration (15 minutes)**

### **Custom Domain Setup:**
```bash
# Add custom domain
aws amplify create-domain-association \
  --app-id YOUR_APP_ID \
  --domain-name "emp-portal.yourdomain.com" \
  --sub-domain-settings '{
    "prefix": "www",
    "branchName": "main"
  }' \
  --region ap-southeast-1
```

### **Build Notifications:**
```bash
# Set up SNS notifications for build status
aws amplify put-webhook \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --description "Build notifications" \
  --region ap-southeast-1
```

### **Access Control:**
```bash
# Enable password protection for staging
aws amplify update-branch \
  --app-id YOUR_APP_ID \
  --branch-name develop \
  --enable-basic-auth \
  --basic-auth-credentials "username:password" \
  --region ap-southeast-1
```

---

## 🎯 **Step 5: Test CI/CD Pipeline (10 minutes)**

### **Trigger First Deployment:**
1. **Make a code change** in your GitLab repository
2. **Commit and push** to main branch
3. **Monitor build** in Amplify console

### **Verify Build Process:**
```bash
# Check build status
aws amplify list-jobs \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --region ap-southeast-1

# Get build details
aws amplify get-job \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --job-id YOUR_JOB_ID \
  --region ap-southeast-1
```

### **Test Deployment:**
- **Production URL:** `https://main.YOUR_APP_ID.amplifyapp.com`
- **Staging URL:** `https://develop.YOUR_APP_ID.amplifyapp.com`

---

## 📋 **Step 6: Configure API Integration**

### **Update Environment Variables:**
```javascript
// In your frontend code
const API_BASE_URL = process.env.REACT_APP_API_GATEWAY_URL || 'https://your-api-gateway-alb.com';

// API calls
const response = await fetch(`${API_BASE_URL}/api/endpoint`);
```

### **CORS Configuration:**
Ensure your API Gateway ALB allows requests from Amplify domains:
```bash
# Update API Gateway CORS settings
# Allow origins: https://main.YOUR_APP_ID.amplifyapp.com
```

---

## 🔄 **CI/CD Workflow**

### **Automatic Pipeline Flow:**
```
GitLab Push → Amplify Detects Change → Build Phase → Test Phase → Deploy Phase → Live
```

### **Build Phases:**
1. **Pre-build:** Install dependencies (`npm ci`)
2. **Build:** Create production bundle (`npm run build`)
3. **Post-build:** Run tests (optional)
4. **Deploy:** Upload to CDN and invalidate cache

### **Branch Strategy:**
- **main branch** → Production deployment
- **develop branch** → Staging deployment  
- **feature branches** → Preview deployments (optional)

---

## 🚨 **Troubleshooting**

### **Common Issues:**

**Build Fails:**
```bash
# Check build logs
aws amplify get-job \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --job-id FAILED_JOB_ID \
  --region ap-southeast-1
```

**Environment Variables Not Working:**
- Ensure variables are prefixed with `REACT_APP_` for React
- Check case sensitivity
- Verify in Amplify console under App Settings → Environment Variables

**API Calls Failing:**
- Check CORS configuration on API Gateway
- Verify API Gateway URL in environment variables
- Check network tab in browser dev tools

---

## ✅ **Success Verification**

### **Checklist:**
- [ ] GitLab repository connected to Amplify
- [ ] Build configuration working (`amplify.yml`)
- [ ] Environment variables configured
- [ ] Production and staging branches deployed
- [ ] Custom domain configured (optional)
- [ ] API integration working
- [ ] Build notifications set up

### **Test Scenarios:**
1. **Push to main** → Production deployment
2. **Push to develop** → Staging deployment
3. **Create PR** → Preview deployment (if enabled)
4. **API calls** → Successful responses from backend

---

## 💰 **Cost Considerations**

### **Amplify Pricing:**
- **Build minutes:** $0.01 per build minute
- **Data transfer:** $0.15 per GB
- **Storage:** $0.023 per GB per month
- **Requests:** $0.30 per million requests

### **Estimated Monthly Cost:**
- **Small app:** $5-15/month
- **Medium app:** $15-50/month
- **Large app:** $50-200/month

---

## 🎯 **Integration with Existing Architecture**

### **Your Complete Setup:**
```
EMP UI (Amplify CI/CD) ← GitLab
     ↓ (API calls)
API Gateway ALB → API Gateway Service → Other Microservices
```

### **Benefits:**
- ✅ **Separate CI/CD** for frontend and backend
- ✅ **Optimized** for frontend deployment
- ✅ **Automatic CDN** distribution
- ✅ **Branch-based** deployments
- ✅ **Built-in monitoring** and logging

---

**🚀 Your EMP UI will have professional-grade CI/CD with minimal setup effort!**
