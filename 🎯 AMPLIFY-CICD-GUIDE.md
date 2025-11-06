# 🎯 AWS Amplify CI/CD Implementation Guide for EMP UI

## 📊 **Overview**

**Goal:** Set up automated CI/CD pipeline for EMP UI frontend using AWS Amplify's built-in capabilities  
**Source:** GitLab repository  
**Target:** Amplify hosting with automatic deployments  
**Timeline:** 30-60 minutes setup  

---

## 🚀 **Step 1: Connect GitLab Repository**

### **🖱️ Console Method (Recommended for Beginners):**

#### **1.1 Access Amplify Console:**
1. **Login to AWS Console** → Search "Amplify" → Click "AWS Amplify"
2. **Click "Get Started"** under "Amplify Hosting"
3. **Select "Deploy without Git provider"** if you want to upload manually, OR
4. **Select "GitLab"** for automatic CI/CD

#### **1.2 Connect GitLab Repository:**
1. **Click "GitLab"** as your Git provider
2. **Click "Continue"**
3. **Authorize AWS Amplify:**
   - You'll be redirected to GitLab
   - Click "Authorize" to allow AWS Amplify access
   - You'll be redirected back to AWS Console

4. **Select Repository:**
   - **Repository:** Select your EMP UI repository from dropdown
   - **Branch:** Select `main` or `master` branch
   - **Click "Next"**

#### **1.3 Configure App Settings:**
1. **App name:** Enter `emp-ui-app`
2. **Environment name:** Enter `production`
3. **Select existing service role** OR **Create new role**
4. **Click "Next"**

### **⌨️ CLI Method (Advanced Users):**
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

## ⚙️ **Step 2: Configure Build Settings**

### **🖱️ Console Method:**

#### **2.1 Review Build Settings:**
1. **Amplify will auto-detect** your frontend framework (React, Vue, Angular, etc.)
2. **Review the suggested build commands:**
   - **Build command:** `npm run build`
   - **Base directory:** `/` (root of repository)
   - **Build output directory:** `build` or `dist`

#### **2.2 Customize Build Settings (if needed):**
1. **Click "Edit"** next to build settings
2. **Modify build specification:**
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
       baseDirectory: build
       files:
         - '**/*'
     cache:
       paths:
         - node_modules/**/*
   ```
3. **Click "Save"**

#### **2.3 Environment Variables:**
1. **Click "Environment variables"** tab
2. **Click "Add variable"**
3. **Add variables:**
   - **Key:** `REACT_APP_API_GATEWAY_URL`
   - **Value:** `https://your-api-gateway-alb-dns-name`
4. **Click "Save"**

### **⌨️ CLI Method:**
```bash
# Set environment variables via CLI
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

## 🌿 **Step 3: Configure Branch Deployment**

### **🖱️ Console Method:**

#### **3.1 Review and Deploy:**
1. **Review all settings** on the final page
2. **Click "Save and deploy"**
3. **Wait for initial deployment** (5-10 minutes)

#### **3.2 Add Additional Branches:**
1. **Go to your Amplify app** → Click on app name
2. **Click "Connect branch"** button
3. **Select branch:** Choose `develop` or `staging`
4. **Configure settings:**
   - **Environment:** `staging`
   - **Build settings:** Same as production or customize
5. **Click "Save and deploy"**

#### **3.3 Branch Settings:**
1. **Click on branch name** (e.g., `main`)
2. **Configure branch settings:**
   - **Auto build:** Toggle ON for automatic deployments
   - **Pull request previews:** Toggle ON for PR previews
   - **Environment variables:** Add branch-specific variables

### **⌨️ CLI Method:**
```bash
# Create production branch deployment
aws amplify create-branch \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --framework "React" \
  --enable-auto-build \
  --region ap-southeast-1

# Create staging branch deployment
aws amplify create-branch \
  --app-id YOUR_APP_ID \
  --branch-name develop \
  --framework "React" \
  --enable-auto-build \
  --region ap-southeast-1
```

---

## 🔧 **Step 4: Advanced Configuration**

### **🖱️ Console Method:**

#### **4.1 Custom Domain Setup:**
1. **Go to App settings** → **Domain management**
2. **Click "Add domain"**
3. **Enter domain:** `emp-portal.yourdomain.com`
4. **Configure subdomains:**
   - **Subdomain:** `www` → **Branch:** `main`
   - **Subdomain:** `staging` → **Branch:** `develop`
5. **Click "Save"**
6. **Wait for SSL certificate** provisioning (15-30 minutes)

#### **4.2 Access Control (Password Protection):**
1. **Go to App settings** → **Access control**
2. **Select branch** (e.g., `develop` for staging)
3. **Toggle "Restrict access"** ON
4. **Set credentials:**
   - **Username:** `staging`
   - **Password:** `your-secure-password`
5. **Click "Save"**

#### **4.3 Build Notifications:**
1. **Go to App settings** → **Notifications**
2. **Click "Add notification"**
3. **Configure notification:**
   - **Event:** Build succeeds/fails
   - **Target:** Email or SNS topic
   - **Recipients:** Your email address
4. **Click "Save"**

#### **4.4 Monitoring and Logs:**
1. **Go to Monitoring** tab
2. **View build history** and deployment status
3. **Click on build** to see detailed logs
4. **Monitor performance** metrics

### **⌨️ CLI Method:**
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

# Enable password protection for staging
aws amplify update-branch \
  --app-id YOUR_APP_ID \
  --branch-name develop \
  --enable-basic-auth \
  --basic-auth-credentials "username:password" \
  --region ap-southeast-1
```

---

## 🎯 **Step 5: Test CI/CD Pipeline**

### **🖱️ Console Method:**

#### **5.1 Monitor First Deployment:**
1. **Go to your Amplify app** in console
2. **Click on branch** (e.g., `main`)
3. **Watch build progress:**
   - **Provision:** Setting up build environment
   - **Build:** Running build commands
   - **Deploy:** Uploading to CDN
   - **Verify:** Final checks

#### **5.2 Test Manual Deployment:**
1. **Click "Redeploy this version"** button
2. **Monitor build logs** in real-time
3. **Check deployment status**

#### **5.3 Test Automatic Deployment:**
1. **Make a code change** in GitLab
2. **Commit and push** to main branch
3. **Go to Amplify console**
4. **Watch automatic build** trigger
5. **Monitor build progress**

#### **5.4 Access Your Application:**
1. **Copy the app URL** from Amplify console
2. **Format:** `https://main.d1234567890.amplifyapp.com`
3. **Test in browser**
4. **Verify API calls** work correctly

### **⌨️ CLI Method:**
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

# Start manual deployment
aws amplify start-job \
  --app-id YOUR_APP_ID \
  --branch-name main \
  --job-type RELEASE \
  --region ap-southeast-1
```

---

## 📱 **Console Navigation Quick Reference**

### **Main Amplify Console Sections:**
1. **All apps** - List of all Amplify applications
2. **App dashboard** - Overview of specific app
3. **Branches** - Manage branch deployments
4. **Deployments** - Build history and logs
5. **Monitoring** - Performance metrics
6. **App settings** - Configuration options

### **Key Console Actions:**
- **Deploy branch:** Manual deployment trigger
- **Connect branch:** Add new branch for deployment
- **Edit build settings:** Modify build configuration
- **Add domain:** Custom domain setup
- **Set environment variables:** App configuration
- **View logs:** Troubleshoot build issues

### **Console URLs:**
- **Main Console:** `https://console.aws.amazon.com/amplify/`
- **App Dashboard:** `https://console.aws.amazon.com/amplify/home#/YOUR_APP_ID`
- **Build Logs:** `https://console.aws.amazon.com/amplify/home#/YOUR_APP_ID/YourBranch/deployments`

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
