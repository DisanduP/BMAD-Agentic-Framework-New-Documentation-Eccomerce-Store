# BMAD-Agentic-Framework-New-Documentation-Eccomerce-Store
BMAD Agentic Framework New Documentation Eccomerce Store I Have Created.

# Ecommerce Store Deployment Guide

This document outlines the complete process of building and deploying a full-stack ecommerce application to Azure, including backend API on Azure VM and frontend on Azure Static Web Apps.

## Overview

- **Backend**: Node.js/Express API deployed on Azure Ubuntu VM
- **Frontend**: React/Vite SPA deployed on Azure Static Web Apps
- **Database**: (Assumed to be configured separately)
- **CI/CD**: GitHub Actions for automated frontend deployment

## Prerequisites

- Azure account with active subscription
- GitHub account
- Node.js 20+ installed locally
- Azure CLI installed
- Git configured

## Step 1: Backend Deployment on Azure VM

### 1.1 Create Azure VM
```bash
az group create --name ECOMMERCE-RG --location 'East US'
az vm create \
  --resource-group ECOMMERCE-RG \
  --name ecommerce-vm \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --size Standard_B2s \
  --public-ip-sku Standard
```

### 1.2 Configure VM Security
```bash
# Open ports for SSH, HTTP, and API
az vm open-port --resource-group ECOMMERCE-RG --name ecommerce-vm --port 22
az vm open-port --resource-group ECOMMERCE-RG --name ecommerce-vm --port 80
az vm open-port --resource-group ECOMMERCE-RG --name ecommerce-vm --port 3001
```

### 1.3 Deploy Backend Code
```bash
# SSH into VM
ssh azureuser@<VM_PUBLIC_IP>

# Install Node.js and npm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clone/upload your backend code
git clone <your-backend-repo> ecommerce-store
cd ecommerce-store/ecommerce-backend

# Install dependencies and build
npm install
npm run build

# Start the server (configure PM2 or systemd for production)
npm start
```

### 1.4 Verify Backend
```bash
# Test API endpoint
curl http://<VM_PUBLIC_IP>:3001/api/health
```

## Step 2: Frontend Development and Build

### 2.1 Frontend Setup
- React 19.1.1 with Vite
- TypeScript, Tailwind CSS, React Router
- Axios for API calls

### 2.2 Configure API Base URL
Update `src/services/api.ts`:
```typescript
const API_BASE_URL = 'http://<VM_PUBLIC_IP>:3001/api';
```

### 2.3 Build Frontend
```bash
npm install
npm run build
```

## Step 3: Frontend Deployment to Azure Static Web Apps

### 3.1 Create Dedicated Frontend Repository
```bash
# Create new GitHub repo: ecommerce-frontend-deploy
# Push only frontend code (exclude backend)
git init
git remote add origin https://github.com/DisanduP/ecommerce-frontend-deploy.git
git add .
git commit -m "Initial frontend deployment"
git push -u origin master
```

### 3.2 Create Azure Static Web App
```bash
az staticwebapp create \
  --name ecommerce-frontend \
  --resource-group ECOMMERCE-RG \
  --source https://github.com/DisanduP/ecommerce-frontend-deploy \
  --branch master \
  --location 'East US' \
  --login-with-github \
  --output json
```

### 3.3 Configure GitHub Actions Workflow
Create `.github/workflows/azure-static-web-apps-[app-name].yml`:

```yaml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - master
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - master

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name != github.repository)
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          lfs: true
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_[APP_NAME] }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          app_location: "/"
          api_location: ""
          output_location: "dist"

  close_pull_request_job:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request Job
    steps:
      - name: Close Pull Request
        id: closepullrequest
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_[APP_NAME] }}
          action: "close"
```

### 3.4 Configure Package.json for Deployment
Ensure `package.json` has:
```json
{
  "engines": {
    "node": ">=20.0.0"
  }
}
```

### 3.5 Get and Set Deployment Token

1. Go to Azure Portal → Static Web Apps → [your app]
2. Settings → Deployment token
3. Regenerate if needed, copy the token
4. Add to GitHub repo secrets:
   - Name: `AZURE_STATIC_WEB_APPS_API_TOKEN_[APP_NAME]`
   - Value: The token

## Issues Encountered and Solutions

### Issue 1: VM Frontend Serving Issues
**Problem**: Express server failed to serve built React app on VM.
**Solution**: Switched to Azure Static Web Apps for better SPA hosting.

### Issue 2: Node Version Mismatch
**Problem**: Oryx used Node 18, but Vite requires Node 20+.
**Solution**: Specified `engines: {"node": ">=20.0.0"}` in package.json and used Node 20 in workflow.

### Issue 3: Oryx Build Failures
**Problem**: Oryx couldn't determine app artifact location.
**Solution**: Set `output_location: "dist"` in workflow to specify built files location.

### Issue 4: Invalid Workflow Parameters
**Problem**: Used `deployment_token` instead of `azure_static_web_apps_api_token`.
**Solution**: Corrected parameter name in workflow YAML.

### Issue 5: Missing Deployment Token
**Problem**: Token not set in GitHub secrets.
**Solution**: Regenerate token in Azure Portal and add to GitHub secrets.

## Final Architecture

```
┌─────────────────┐    ┌─────────────────┐
│   Azure VM      │    │ Azure Static   │
│   Backend API   │◄──►│   Web Apps     │
│   Port 3001     │    │   Frontend     │
└─────────────────┘    └─────────────────┘
         ▲                       ▲
         │                       │
    ┌────┴───────────────────────┴────┐
    │         Database               │
    └─────────────────────────────────┘
```

## URLs

- **Backend API**: http://<VM_PUBLIC_IP>:3001/api
- **Frontend**: https://[app-name].azurestaticapps.net

## Monitoring and Maintenance

- Monitor GitHub Actions for deployment status
- Check Azure Portal for resource usage and costs
- Update dependencies regularly
- Configure proper logging and error handling

## Cost Optimization

- Use B-series VMs for development
- Set up auto-shutdown for VMs when not in use
- Monitor Static Web Apps usage (free tier available)

## Security Considerations

- Use HTTPS for production
- Implement proper CORS settings
- Secure API endpoints with authentication
- Regularly update dependencies
- Use Azure Key Vault for secrets

## Next Steps

- Add database integration
- Implement user authentication
- Set up monitoring and logging
- Configure custom domain
- Add CI/CD for backend updates
