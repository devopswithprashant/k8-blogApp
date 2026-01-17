# K8-BlogApp: Kubernetes GitOps Repository

A production-grade, custom GitOps pipeline for deploying a multi-tier blog application across multiple Kubernetes environments (dev, qa, stage, prod) on AWS EKS using GitHub Actions and AWS Systems Manager.

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Directory Structure](#directory-structure)
- [Technical Workflow](#technical-workflow)
- [User Manual](#user-manual)
- [Setup & Prerequisites](#setup--prerequisites)
- [Deployment Process](#deployment-process)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Security Considerations](#security-considerations)
- [FAQ](#faq)

---

## 🎯 Project Overview

**K8-BlogApp** is an in-house GitOps solution that automates Kubernetes deployments without relying on external GitOps operators like ArgoCD. It provides:

- **Declarative Infrastructure**: All Kubernetes manifests stored in Git
- **Automated Deployments**: Push-triggered CI/CD via GitHub Actions
- **Multi-Environment Support**: Separate configurations for dev, qa, stage, and prod
- **AWS Integration**: Uses AWS Systems Manager (SSM) for secure deployment execution
- **Health Validation**: Automatic health checks post-deployment
- **Audit Trail**: Full deployment history via AWS CloudTrail

**Key Benefits:**
- ✅ Single source of truth (Git)
- ✅ Auditability via AWS SSM
- ✅ Production safety gates (main branch only)
- ✅ No external cluster dependencies
- ✅ Cost-effective (no additional operators needed)

---

## 🏗️ Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  apps/blog/{dev|qa|stage|prod}/                             │   │
│  │  ├── backend/                                                │   │
│  │  │   ├── blog-deployment.yaml                               │   │
│  │  │   └── blog-service.yaml                                  │   │
│  │  ├── frontend/                                               │   │
│  │  │   ├── blog-ui-deployment.yaml                            │   │
│  │  │   └── blog-ui-service.yaml                               │   │
│  │  ├── namespace/namespace.yaml                               │   │
│  │  └── networking/frontend-to-backend.yaml                    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              │ (git push)                           │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │     GitHub Actions Workflow: deploy.yaml                    │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │ 1. AWS Credentials via OIDC                         │   │   │
│  │  │ 2. Change Detection (git diff)                      │   │   │
│  │  │ 3. Extract Environment Directory                    │   │   │
│  │  │ 4. Production Safety Gate Check                     │   │   │
│  │  │ 5. Loop: For each changed env:                      │   │   │
│  │  │    - Trigger AWS SSM send-command                  │   │   │
│  │  │    - Wait for deployment completion                │   │   │
│  │  │    - Validate health check results                 │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        AWS Account                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │   AWS Systems Manager (SSM)                                 │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │ Document: K8SDeployer                               │   │   │
│  │  │ Parameters:                                          │   │   │
│  │  │  - awsRegion: us-east-1                            │   │   │
│  │  │  - clusterName: blogapp-cluster (or prod)          │   │   │
│  │  │  - manifestDirectory: apps/blog/dev (etc)          │   │   │
│  │  │  - k8namespace: dwp-dev (extracted from manifest) │   │   │
│  │  │  - gitRef: feature-branch (git commit ref)         │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              │ (send-command)                       │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │   EKS Bastion Host (tagged: Name=eks-bastion-blogapp)       │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │ Execute K8SDeployer Document:                       │   │   │
│  │  │                                                      │   │   │
│  │  │ 1. git clone k8-blogApp repo                        │   │   │
│  │  │ 2. git checkout <gitRef>                            │   │   │
│  │  │ 3. cd apps/blog/dev                                │   │   │
│  │  │ 4. kubectl apply -f . -n dwp-dev                   │   │   │
│  │  │ 5. Health Check:                                   │   │   │
│  │  │    - kubectl rollout status deployment/...         │   │   │
│  │  │    - Verify pod readiness                          │   │   │
│  │  │ 6. Return status (Success/Failed)                  │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │    EKS Cluster: blogapp-cluster                            │   │
│  │                                                              │   │
│  │    Namespace: dwp-dev                                       │   │
│  │    ├── Deployment: blog-backend (2 replicas)               │   │
│  │    │   ├── Pod 1: Running (v1.0.11)                       │   │
│  │    │   └── Pod 2: Running (v1.0.11)                       │   │
│  │    ├── Deployment: blog-frontend (2 replicas)              │   │
│  │    │   ├── Pod 1: Running (1.1.7)                        │   │
│  │    │   └── Pod 2: Running (1.1.7)                        │   │
│  │    ├── Service: blog-service (ClusterIP)                   │   │
│  │    ├── Service: blog-ui-service (ClusterIP)                │   │
│  │    └── NetworkPolicy: allow-frontend-to-backend            │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
Developer                GitHub              GitHub Actions         AWS SSM            EKS Bastion        EKS Cluster
    │                       │                      │                    │                  │                   │
    │─ git push manifest ──>│                      │                    │                  │                   │
    │                       │──(webhook)──────────>│                    │                  │                   │
    │                       │                      │─(aws oidc)────────>│                  │                   │
    │                       │                      │                    │                  │                   │
    │                       │                      │<──(creds)─────────│                  │                   │
    │                       │                      │                    │                  │                   │
    │                       │                      │───(change detect)──>                  │                   │
    │                       │                      │                    │                  │                   │
    │                       │                      │──(send-command)────>send K8SDeployer  │                   │
    │                       │                      │                    │──────────────────>git clone           │
    │                       │                      │                    │                  │ git checkout      │
    │                       │                      │                    │                  │ kubectl apply     │
    │                       │                      │                    │                  │ health check      │
    │                       │                      │                    │                  │─────────────────>Deploy
    │                       │                      │                    │                  │                   │Pods Running
    │                       │                      │<──(command output)─│                  │                   │
    │                       │                      │                    │                  │                   │
    │                       │                      │◄──(status=Success)──                  │                   │
    │                       │                      │                    │                  │                   │
    │<─── deployment ✅ ────────────────────────────                   │                  │                   │
    │
```

---

## 📁 Directory Structure

```
k8-blogApp/
├── README.md                          # This file
├── LICENSE                            # License information
├── .github/
│   └── workflows/
│       └── deploy.yaml                # Main CI/CD pipeline
│
└── apps/
    └── blog/                          # Application name
        ├── dev/                       # Development environment
        │   ├── backend/
        │   │   ├── blog-deployment.yaml
        │   │   └── blog-service.yaml
        │   ├── frontend/
        │   │   ├── blog-ui-deployment.yaml
        │   │   └── blog-ui-service.yaml
        │   ├── namespace/
        │   │   └── namespace.yaml
        │   └── networking/
        │       └── frontend-to-backend.yaml
        │
        ├── qa/                        # QA environment
        │   ├── backend/
        │   ├── frontend/
        │   ├── namespace/
        │   └── networking/
        │
        ├── stage/                     # Staging environment
        │   ├── backend/
        │   ├── frontend/
        │   ├── namespace/
        │   └── networking/
        │
        └── prod/                      # Production environment
            ├── backend/
            ├── frontend/
            ├── namespace/
            └── networking/
```

### Directory Conventions

- **apps/**: Root folder for all applications
- **apps/<app-name>/**: Specific application (e.g., `blog`)
- **apps/<app-name>/<environment>/**: Environment-specific manifests
  - **environment**: One of `dev`, `qa`, `stage`, or `prod`
  - **backend/**: Backend application Kubernetes manifests
  - **frontend/**: Frontend application Kubernetes manifests
  - **namespace/**: Kubernetes namespace definition
  - **networking/**: Network policies and service mesh configs

---

## 🔄 Technical Workflow

### Complete Deployment Flow

#### **Phase 1: Trigger Detection (GitHub Actions)**

1. Developer pushes changes to GitHub
2. Webhook triggers `deploy.yaml` workflow
3. Workflow runs on `push` event to `apps/**` paths only
4. AWS credentials obtained via OIDC (no long-lived secrets)

#### **Phase 2: Change Analysis**

```bash
# Step: "Determine the change env by before/after SHAs"
```

**Purpose**: Intelligently detect which environment(s) changed

**Logic**:
```
Input: git diff output (raw file paths)
       ↓
Parse: Extract environment from file path
       apps/blog/dev/backend/blog-deployment.yaml → apps/blog/dev
       ↓
Filter: Keep only valid environments (dev|qa|stage|prod)
        Skip non-blog apps
        ↓
Deduplicate: Remove duplicates
             ↓
Output: CHANGED_DIRS environment variable
        Example: apps/blog/dev\napps/blog/qa
```

**Special Cases**:
- **Initial commit**: Uses `git ls-tree` to list all files
- **Feature branch to prod**: Blocked by production safety gate

#### **Phase 3: Production Safety Gate**

```bash
if [ "$envName" == "prod" ] && [ "$REFNAME" != "main" ]; then
  echo "Skipping prod deployment on non-main branch"
  continue
fi
```

**Purpose**: Prevent accidental production deployments from feature branches

**Behavior**:
- ✅ `main` branch → prod deployments allowed
- ❌ Feature branches → prod changes skipped (avoids deployment failure)

#### **Phase 4: SSM Command Execution**

For each detected environment directory:

```bash
# 1. Determine EKS cluster & bastion
if [ "$envName" == "prod" ]; then
  eksBastion="eks-bastion-prod-blogapp"
  eksCluster="blogapp-prod-cluster"
else
  eksBastion="eks-bastion-blogapp"
  eksCluster="blogapp-cluster"
fi

# 2. Extract namespace from manifest
namespaceName=$(grep -E "^[[:space:]]*name:" namespace/namespace.yaml | sed 's/.*name:[[:space:]]*//' | head -n1)

# 3. Send SSM command
aws ssm send-command \
  --document-name "K8SDeployer" \
  --targets Key=tag:Name,Values=${eksBastion} \
  --parameters awsRegion=us-east-1,clusterName=${eksCluster},gitRef=${{ github.ref_name }},manifestDirectory=${changeDir},k8namespace=${namespaceName}
```

**Parameters passed to SSM K8SDeployer document**:

| Parameter | Value | Example |
|-----------|-------|---------|
| `awsRegion` | AWS region | `us-east-1` |
| `clusterName` | EKS cluster name | `blogapp-cluster` |
| `gitRef` | Git branch/commit | `feature/add-caching` |
| `manifestDirectory` | Path in repo | `apps/blog/dev` |
| `k8namespace` | K8s namespace | `dwp-dev` |

#### **Phase 5: Execution on Bastion (SSM Document)**

The `K8SDeployer` SSM document executes on the bastion host:

```bash
#!/bin/bash
# 1. Clone repository
git clone https://github.com/org/k8-blogApp.git
cd k8-blogApp
git checkout <gitRef>

# 2. Navigate to manifest directory
cd <manifestDirectory>  # e.g., apps/blog/dev

# 3. Apply manifests
kubectl apply -f . -n <k8namespace>

# 4. Health check
for deployment in $(kubectl get deployments -n <k8namespace> -o jsonpath='{.items[*].metadata.name}'); do
  kubectl rollout status deployment/$deployment -n <k8namespace> --timeout=300s
  if [ $? -ne 0 ]; then
    echo "Deployment failed"
    exit 1
  fi
done

# 5. Return status
echo "✅ Deployment successful"
```

#### **Phase 6: Status Polling & Reporting**

GitHub Actions polls SSM for command status:

```bash
STATUS="InProgress"
while [[ "$STATUS" == "InProgress" || "$STATUS" == "Pending" ]]; do
  sleep 5  # Poll every 5 seconds
  STATUS=$(aws ssm list-command-invocations --command-id $COMMAND_ID --query "CommandInvocations[0].Status" --output text)
  echo "Current status: $STATUS"
done

# Capture output
STDOUT=$(aws ssm list-command-invocations --command-id $COMMAND_ID --details --query "CommandInvocations[0].CommandPlugins[0].Output" --output text)

if [ "$STATUS" != "Success" ]; then
  echo "Deployment failed"
  exit 1
else
  echo "Deployment succeeded"
fi
```

#### **Phase 7: Completion**

- ✅ All deployments successful → Workflow passes
- ❌ Any deployment failed → Workflow fails with exit code 1
- 📊 Results visible in GitHub Actions UI

---

## 📖 User Manual

### How to Deploy Your Changes

#### **Scenario 1: Deploy to Dev Environment**

**Objective**: Test your changes in the dev environment

**Steps**:

1. **Clone the repository**
   ```bash
   git clone https://github.com/org/k8-blogApp.git
   cd k8-blogApp
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/update-blog-backend
   ```

3. **Make changes to dev manifests**
   ```bash
   # Edit deployment image version
   vim apps/blog/dev/backend/blog-deployment.yaml
   
   # Example change:
   # Change: image: ...blog:v1.0.11
   # To:     image: ...blog:v1.0.12
   ```

4. **Commit and push**
   ```bash
   git add apps/blog/dev/
   git commit -m "chore: update blog backend to v1.0.12 in dev"
   git push origin feature/update-blog-backend
   ```

5. **Monitor deployment**
   - Go to GitHub → Actions tab
   - Click on your workflow run (named "Deploy")
   - Watch for ✅ or ❌ status
   - Expected output:
     ```
     ✅ Deployment succeeded for apps/blog/dev
     ```

6. **Verify in cluster**
   ```bash
   # SSH to EKS bastion
   ssh ec2-user@eks-bastion-blogapp
   
   # Check deployment status
   kubectl rollout status deployment/blog-backend -n dwp-dev
   kubectl get pods -n dwp-dev
   ```

---

#### **Scenario 2: Deploy to Multiple Environments**

**Objective**: Update the same app across dev, qa, and stage

**Steps**:

1. **Update all three environment manifests**
   ```bash
   git checkout -b feature/multi-env-update
   
   # Update dev
   sed -i 's/blog:v1.0.11/blog:v1.0.12/g' apps/blog/dev/backend/blog-deployment.yaml
   
   # Update qa
   sed -i 's/blog:v1.0.11/blog:v1.0.12/g' apps/blog/qa/backend/blog-deployment.yaml
   
   # Update stage
   sed -i 's/blog:v1.0.11/blog:v1.0.12/g' apps/blog/stage/backend/blog-deployment.yaml
   ```

2. **Commit changes**
   ```bash
   git add apps/blog/
   git commit -m "chore: update blog backend to v1.0.12 in dev, qa, and stage"
   git push origin feature/multi-env-update
   ```

3. **Monitor all deployments**
   - Workflow will detect 3 changed environments
   - Will deploy sequentially to all three
   - Expected output:
     ```
     ✅ Deployment succeeded for apps/blog/dev
     ✅ Deployment succeeded for apps/blog/qa
     ✅ Deployment succeeded for apps/blog/stage
     All non-prod deployments (if any) completed.
     ```

---

#### **Scenario 3: Deploy to Production (main branch only)**

**Objective**: Deploy to production from main branch

**Requirements**:
- ✅ Changes committed to `main` branch
- ✅ All tests passing
- ✅ Peer review completed
- ✅ Change is in `apps/blog/prod/*`

**Steps**:

1. **Ensure you're on main**
   ```bash
   git checkout main
   git pull origin main
   ```

2. **Update production manifests**
   ```bash
   # Direct edit (in real scenario, this would be via PR + merge)
   sed -i 's/blog:v1.0.11/blog:v1.0.12/g' apps/blog/prod/backend/blog-deployment.yaml
   git add apps/blog/prod/
   git commit -m "chore: update blog backend to v1.0.12 in prod"
   git push origin main
   ```

3. **Verify deployment**
   - Production deployments use separate bastion: `eks-bastion-prod-blogapp`
   - And separate cluster: `blogapp-prod-cluster`
   - Expected output:
     ```
     ✅ Deployment succeeded for apps/blog/prod
     ```

---

#### **Scenario 4: Attempted Prod Deployment from Feature Branch (Blocked)**

**Objective**: Understand safety mechanism

**Steps**:

1. **Attempt to change prod from feature branch**
   ```bash
   git checkout -b feature/prod-hotfix
   sed -i 's/v1.0.11/v1.0.13/g' apps/blog/prod/backend/blog-deployment.yaml
   git add apps/blog/prod/
   git commit -m "hotfix: emergency update to prod"
   git push origin feature/prod-hotfix
   ```

2. **Observe workflow behavior**
   - Workflow detects prod changes on non-main branch
   - Skips SSM deployment with message:
     ```
     Detected prod change on non-main branch — skipping SSM for apps/blog/prod
     ```
   - ✅ Workflow passes (no deployment attempted)
   - 🛡️ Production protected from accidental changes

3. **Proper way to deploy prod**
   ```bash
   # Create PR → get approvals → merge to main
   git checkout main
   git pull origin main
   # Now make your prod changes
   sed -i 's/v1.0.11/v1.0.13/g' apps/blog/prod/backend/blog-deployment.yaml
   git add apps/blog/prod/
   git commit -m "hotfix: emergency update to prod"
   git push origin main
   # Now deployment happens automatically
   ```

---

### Image Tag Auto-Update Flow

**Current Manual Process**:

When you need to deploy a new image version:

1. **New image is built and pushed to ECR**
   ```bash
   # Build and push happens in your app CI/CD pipeline
   docker build -t blog:v1.0.12 .
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 850756192530.dkr.ecr.us-east-1.amazonaws.com
   docker tag blog:v1.0.12 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:v1.0.12
   docker push 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:v1.0.12
   ```

2. **Update manifests in this repository**
   ```bash
   # Manually update the YAML files
   git clone https://github.com/org/k8-blogApp.git
   git checkout -b update-blog-image-v1.0.12
   
   # Update all environments
   for env in dev qa stage prod; do
     sed -i 's|blog:v[0-9.]*|blog:v1.0.12|g' \
       apps/blog/$env/backend/blog-deployment.yaml
   done
   
   git add apps/blog/
   git commit -m "chore: bump blog backend to v1.0.12"
   git push origin update-blog-image-v1.0.12
   ```

3. **Merge to main or feature branch**
   - Create PR
   - Get approval
   - Merge to target branch

4. **Deployment triggers automatically**
   - If merged to `main` → production deploys
   - If merged to feature branch → non-prod environments deploy

---

### Health Check Validation

**What happens during health check**:

```bash
# 1. Wait for deployment rollout
kubectl rollout status deployment/blog-backend -n dwp-dev --timeout=300s

# 2. Verify replicas are ready
kubectl get deployment blog-backend -n dwp-dev -o jsonpath='{.status.readyReplicas}'
# Should match: spec.replicas (e.g., 2 == 2)

# 3. Check pod readiness
kubectl get pods -n dwp-dev -l app=blog-backend
# All pods should be in Running state

# 4. Verify service endpoints
kubectl get endpoints blog-service -n dwp-dev
# Should have IP addresses listed
```

**Health check failure scenarios**:

| Scenario | Cause | Resolution |
|----------|-------|------------|
| Pods stuck in Pending | Insufficient resources | Scale down replicas or add nodes |
| ImagePullBackOff | Image tag invalid | Verify image exists in ECR |
| CrashLoopBackOff | App startup error | Check pod logs: `kubectl logs <pod>` |
| Deployment timeout | Slow pod startup | Increase timeout in SSM document |

---

## ⚙️ Setup & Prerequisites

### Prerequisites

**Required Access**:
- [ ] GitHub repository access (push permissions)
- [ ] AWS account access
- [ ] EKS cluster access via bastion host
- [ ] AWS IAM role with SSM permissions

**Required AWS Resources**:
- [ ] EKS cluster (e.g., `blogapp-cluster`)
- [ ] EKS bastion host (tagged as `eks-bastion-blogapp`)
- [ ] SSM document named `K8SDeployer`
- [ ] AWS IAM role for GitHub Actions OIDC
- [ ] ECR repository for images

**Required GitHub Secrets**:
- [ ] `AWS_SSM_ROLE`: ARN of IAM role for OIDC

### Initial Setup

#### **Step 1: Configure AWS OIDC for GitHub Actions**

```bash
# Create IAM OIDC provider (one-time setup)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --client-id-list sts.amazonaws.com

# Create IAM role for GitHub Actions
aws iam create-role \
  --role-name github-actions-k8-blogapp \
  --assume-role-policy-document file://trust-policy.json

# Attach policy (see below)
aws iam put-role-policy \
  --role-name github-actions-k8-blogapp \
  --policy-name ssm-deployment \
  --policy-document file://policy.json
```

**trust-policy.json**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/k8-blogApp:*"
        }
      }
    }
  ]
}
```

**policy.json**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand",
        "ssm:ListCommandInvocations",
        "ssm:GetCommandInvocation"
      ],
      "Resource": "*"
    }
  ]
}
```

#### **Step 2: Create GitHub Secret**

1. Go to GitHub → Repository Settings → Secrets and Variables → Actions
2. Create new secret `AWS_SSM_ROLE`
3. Value: `arn:aws:iam::AWS_ACCOUNT_ID:role/github-actions-k8-blogapp`

#### **Step 3: Create SSM Document**

```bash
aws ssm create-document \
  --name K8SDeployer \
  --document-type Command \
  --content file://ssm-document-k8sdeployer.yaml
```

**ssm-document-k8sdeployer.yaml**:
```yaml
schemaVersion: '2.2'
description: Deploy Kubernetes manifests to EKS cluster
parameters:
  awsRegion:
    type: String
    description: AWS region
  clusterName:
    type: String
    description: EKS cluster name
  gitRef:
    type: String
    description: Git branch or commit SHA
  manifestDirectory:
    type: String
    description: 'Path in repo (e.g., apps/blog/dev)'
  k8namespace:
    type: String
    description: Kubernetes namespace
mainSteps:
  - action: aws:runShellScript
    name: DeployKubernetesManifests
    inputs:
      runCommand:
        - |
          #!/bin/bash
          set -euo pipefail
          
          echo "=== Kubernetes Deployment Started ==="
          echo "Cluster: {{ clusterName }}"
          echo "Namespace: {{ k8namespace }}"
          echo "Directory: {{ manifestDirectory }}"
          
          # Step 1: Clone and checkout
          mkdir -p /tmp/k8s-deploy-${RANDOM}
          cd /tmp/k8s-deploy-${RANDOM}
          
          echo "Cloning repository..."
          git clone https://github.com/YOUR_ORG/k8-blogApp.git
          cd k8-blogApp
          git checkout {{ gitRef }}
          
          # Step 2: Update kubeconfig
          echo "Updating kubeconfig..."
          aws eks update-kubeconfig \
            --region {{ awsRegion }} \
            --name {{ clusterName }}
          
          # Step 3: Validate manifests
          echo "Validating manifests..."
          kubectl apply -f {{ manifestDirectory }} \
            -n {{ k8namespace }} \
            --dry-run=client \
            -o yaml > /tmp/manifest-dry-run.yaml
          
          # Step 4: Apply manifests
          echo "Applying manifests..."
          kubectl apply -f {{ manifestDirectory }} \
            -n {{ k8namespace }}
          
          # Step 5: Health check
          echo "Running health checks..."
          TIMEOUT=300
          ELAPSED=0
          INTERVAL=5
          
          while [ $ELAPSED -lt $TIMEOUT ]; do
            ALL_READY=true
            
            for deployment in $(kubectl get deployments -n {{ k8namespace }} -o jsonpath='{.items[*].metadata.name}'); do
              READY=$(kubectl get deployment $deployment -n {{ k8namespace }} -o jsonpath='{.status.readyReplicas}')
              DESIRED=$(kubectl get deployment $deployment -n {{ k8namespace }} -o jsonpath='{.spec.replicas}')
              
              echo "Deployment: $deployment - Ready: $READY/$DESIRED"
              
              if [ "$READY" != "$DESIRED" ]; then
                ALL_READY=false
              fi
            done
            
            if [ "$ALL_READY" = "true" ]; then
              echo "✅ All deployments are ready"
              exit 0
            fi
            
            sleep $INTERVAL
            ELAPSED=$((ELAPSED + INTERVAL))
          done
          
          echo "❌ Health check timeout after $TIMEOUT seconds"
          exit 1
```

---

## 🚀 Deployment Process

### Complete Deployment Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. TRIGGER                                                       │
│ Developer pushes to apps/blog/dev/backend/blog-deployment.yaml │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 2. WORKFLOW START                                                │
│ GitHub Actions triggers deploy.yaml workflow                    │
│ Duration: ~2 seconds                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 3. AUTHENTICATION                                                │
│ AWS OIDC token obtained                                         │
│ Credentials valid for 1 hour                                    │
│ Duration: ~1 second                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 4. CHANGE DETECTION                                              │
│ Git diff between commits                                         │
│ Extract: apps/blog/dev                                          │
│ Namespace: dwp-dev                                              │
│ Duration: ~2 seconds                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 5. SAFETY GATE CHECK                                             │
│ Is this prod change on non-main? → YES → SKIP                   │
│ Is this prod change on main? → YES → CONTINUE                   │
│ Duration: <1 second                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 6. SSM COMMAND SEND                                              │
│ Document: K8SDeployer                                           │
│ Target: eks-bastion-blogapp                                     │
│ Parameters sent (5 parameters)                                  │
│ Duration: ~1 second                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 7. BASTION EXECUTION (K8SDeployer)                              │
│ Clone repo                        Duration: ~5 seconds          │
│ Checkout branch                   Duration: ~1 second          │
│ kubectl apply                     Duration: ~2 seconds          │
│ Rollout status (wait for pods)    Duration: 10-30 seconds      │
│ Total:                            Duration: ~18-39 seconds      │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 8. STATUS POLLING                                                │
│ GitHub Actions polls SSM every 5 seconds                        │
│ Waits for command completion                                    │
│ Duration: ~5-40 seconds (depends on step 7)                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 9. RESULT PROCESSING                                             │
│ Capture output from SSM                                         │
│ Parse status (Success/Failed)                                   │
│ Duration: ~2 seconds                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│ 10. COMPLETION                                                   │
│ Workflow marked: ✅ SUCCESS or ❌ FAILURE                       │
│ GitHub commit shows deployment status                           │
│ Duration: <1 second                                             │
└────────────────────────────────────────────────────────────────┘

TOTAL TIME: 40-100 seconds depending on pod startup time
```

### Monitoring Deployment Progress

#### **In GitHub Actions UI**:

1. Go to **Actions** tab
2. Click on your workflow run
3. Click on **Deploy-K8** job
4. Watch real-time logs:

```
Step 1: Configure AWS Credentials ✅
Step 2: Checkout                  ✅
Step 3: Determine the change env  ✅
  Detected changed_dirs (canonical):
  apps/blog/dev
  
Step 4: AWS Send Command (loop)   ⏳
  Processing changeDir: apps/blog/dev
  Triggering SSM for apps/blog/dev
  Command sent. Command ID: abc123def456
  Waiting for command to complete...
  Current status for apps/blog/dev: InProgress
  Current status for apps/blog/dev: InProgress
  Current status for apps/blog/dev: Success
  
  Output Log:
  === Kubernetes Deployment Started ===
  Cloning repository...
  Checking out develop...
  Updating kubeconfig...
  Validating manifests...
  Applying manifests...
  Running health checks...
  Deployment: blog-backend - Ready: 2/2
  Deployment: blog-frontend - Ready: 2/2
  ✅ All deployments are ready
  
  ✅ Deployment succeeded for apps/blog/dev

Step 5: Completion                ✅
  All non-prod deployments (if any) completed.
```

#### **In EKS Cluster**:

```bash
# SSH to bastion
ssh ec2-user@eks-bastion-blogapp

# Watch pods being deployed
kubectl get pods -n dwp-dev -w

# Check deployment status
kubectl rollout status deployment/blog-backend -n dwp-dev
kubectl rollout status deployment/blog-frontend -n dwp-dev

# View recent events
kubectl get events -n dwp-dev --sort-by='.lastTimestamp'

# Check service endpoints
kubectl get endpoints -n dwp-dev
```

---

## 🔧 Troubleshooting

### Deployment Failed: "Detected prod change on non-main branch"

**Problem**: You tried to deploy production changes from a feature branch

**Solution**:
```bash
# 1. Merge your feature branch to main first
git checkout main
git pull origin main
git merge feature/your-feature
git push origin main

# 2. Now prod deployment will automatically trigger
```

---

### Deployment Failed: Command timeout after 300 seconds

**Problem**: Pod startup is taking longer than 5 minutes

**Solutions**:

Option 1 - Increase timeout in SSM document:
```yaml
mainSteps:
  - action: aws:runShellScript
    inputs:
      timeoutSeconds: 600  # Increase from 300 to 600
```

Option 2 - Check why pod is slow:
```bash
ssh ec2-user@eks-bastion-blogapp
kubectl describe pod <pod-name> -n dwp-dev
kubectl logs <pod-name> -n dwp-dev
```

Option 3 - Check resource constraints:
```bash
# Check node resources
kubectl top nodes
kubectl describe nodes

# Increase pod resource requests if needed
# Edit apps/blog/dev/backend/blog-deployment.yaml
# Add resources section with appropriate requests/limits
```

---

### Deployment Failed: ImagePullBackOff

**Problem**: Kubernetes cannot pull the Docker image from ECR

**Solutions**:

1. **Verify image exists in ECR**:
```bash
aws ecr describe-images \
  --repository-name dwp/blog \
  --region us-east-1
```

2. **Verify image tag is correct**:
```bash
# Check manifest
cat apps/blog/dev/backend/blog-deployment.yaml | grep image:

# Should output something like:
# image: 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:v1.0.11
```

3. **Verify bastion has ECR login**:
```bash
ssh ec2-user@eks-bastion-blogapp

# Check docker credentials
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  850756192530.dkr.ecr.us-east-1.amazonaws.com
```

4. **Check EKS node IAM role**:
```bash
# Nodes need ECR read permissions
# Check: kubectl describe node <node-name>
# Look for IAM role in metadata
```

---

### Deployment Failed: CrashLoopBackOff

**Problem**: Pod starts but immediately crashes

**Solutions**:

1. **Check pod logs**:
```bash
kubectl logs <pod-name> -n dwp-dev
```

2. **Check previous logs** (if pod crashed):
```bash
kubectl logs <pod-name> -n dwp-dev --previous
```

3. **Check pod events**:
```bash
kubectl describe pod <pod-name> -n dwp-dev
```

4. **Verify environment variables**:
```bash
kubectl get deployment blog-backend -n dwp-dev -o yaml | grep -A 20 env:
```

5. **Test locally** (before deploying):
```bash
docker run -it \
  -e SPRING_DATASOURCE_URL="jdbc:postgresql://..." \
  -e SPRING_DATASOURCE_USERNAME="devuser" \
  -e SPRING_DATASOURCE_PASSWORD="devuser" \
  850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:v1.0.11
```

---

### Deployment Failed: Insufficient Resources

**Problem**: Pods remain in `Pending` state due to insufficient CPU/memory

**Solutions**:

1. **Check node resources**:
```bash
kubectl top nodes
kubectl describe nodes
```

2. **Check pod resource requests**:
```bash
kubectl describe pod <pod-name> -n dwp-dev | grep -A 5 Requests
```

3. **Reduce pod replicas** (temporary):
```bash
# Edit apps/blog/dev/backend/blog-deployment.yaml
# Change: replicas: 2 → replicas: 1
git add apps/blog/dev/
git commit -m "temp: reduce replicas to 1"
git push
```

4. **Add worker nodes** (permanent):
```bash
# Scale EKS node group
aws eks update-nodegroup-config \
  --cluster-name blogapp-cluster \
  --nodegroup-name default \
  --scaling-config desiredSize=5,minSize=3,maxSize=10
```

---

### GitHub Actions Workflow Not Triggering

**Problem**: Pushed changes but workflow didn't run

**Check**:

1. **Verify branch protection rules**:
   - Go to Settings → Branches
   - Check if branch requires status checks

2. **Verify push was to correct path**:
   ```bash
   # Workflow only triggers on apps/** changes
   git log --name-only -1
   # Should show files under apps/
   ```

3. **Check GitHub Actions is enabled**:
   - Settings → Actions → General
   - Ensure "Actions permissions" is set correctly

4. **View workflow runs**:
   - Actions tab → Select workflow → See all runs
   - Check if your push appears

---

### AWS SSM Command Failed

**Problem**: "Failed to send SSM command"

**Solutions**:

1. **Verify bastion host is reachable**:
```bash
aws ssm list-managed-instances --region us-east-1
# Should show eks-bastion-blogapp
```

2. **Verify IAM role has SSM permissions**:
```bash
# Check GitHub Actions role policy
aws iam get-role-policy \
  --role-name github-actions-k8-blogapp \
  --policy-name ssm-deployment
```

3. **Check bastion security group**:
```bash
# Bastion should allow outbound to SSM service
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=eks-bastion-sg"
```

4. **View SSM command output**:
```bash
aws ssm list-command-invocations \
  --region us-east-1 \
  --query 'CommandInvocations[?InstanceIds[0]==`i-xxxxx`]'
```

---

### How to Manually Trigger Deployment

If workflow fails and you need to redeploy:

```bash
# Method 1: Rerun failed workflow
# Go to Actions → Click workflow → Re-run jobs

# Method 2: Manual re-push
git commit --allow-empty -m "chore: trigger redeployment"
git push origin main

# Method 3: Manually run SSM command
aws ssm send-command \
  --document-name "K8SDeployer" \
  --targets Key=tag:Name,Values=eks-bastion-blogapp \
  --parameters awsRegion=us-east-1,clusterName=blogapp-cluster,gitRef=main,manifestDirectory=apps/blog/dev,k8namespace=dwp-dev \
  --region us-east-1
```

---

## ✅ Best Practices

### 1. Always Use Feature Branches

```bash
# ❌ DON'T: Push directly to main
git checkout main
git commit -m "Update deployment"
git push origin main

# ✅ DO: Create feature branch
git checkout -b feature/update-deployment
git commit -m "Update deployment"
git push origin feature/update-deployment
# Create PR, get review, then merge
```

**Why**: Allows review before production changes

---

### 2. Commit Messages Should Be Descriptive

```bash
# ❌ BAD
git commit -m "update"

# ✅ GOOD
git commit -m "chore: update blog-backend image to v1.0.12 in dev and qa"
git commit -m "fix: increase blog-backend memory limit to 512Mi"
git commit -m "feat: add network policy for frontend-to-backend communication"
```

**Why**: Audit trail for deployments

---

### 3. Test Changes in Dev First

```bash
# Step 1: Deploy to dev
git checkout -b feature/update-api
sed -i 's/v1.0.11/v1.0.12/g' apps/blog/dev/backend/blog-deployment.yaml
git commit -m "chore: update to v1.0.12 in dev"
git push origin feature/update-api
# Wait for deployment and verify in dev

# Step 2: After dev validation, update qa
git add apps/blog/dev
sed -i 's/v1.0.11/v1.0.12/g' apps/blog/qa/backend/blog-deployment.yaml
git commit -m "chore: update to v1.0.12 in qa"
git push

# Step 3: Repeat for stage, then prod
```

**Why**: Prevents breaking production

---

### 4. Review Manifests Before Pushing

```bash
# Use kubeval to validate
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xf kubeval-linux-amd64.tar.gz
./kubeval apps/blog/dev/backend/blog-deployment.yaml

# Use kube-score for best practices
docker run -v $(pwd):/app zegl/kube-score score /app/apps/blog/dev/backend/blog-deployment.yaml
```

**Why**: Catch errors before deployment

---

### 5. Monitor Deployments Actively

```bash
# Watch pods during deployment
watch 'kubectl get pods -n dwp-dev'

# Check logs after deployment
kubectl logs deployment/blog-backend -n dwp-dev --tail=50 -f

# Monitor resource usage
kubectl top pods -n dwp-dev
```

**Why**: Catch issues early

---

### 6. Keep Image Tags Consistent

**Pattern**:
- **Semantic Versioning** for app releases: `v1.0.12`, `v1.1.0`, `v2.0.0`
- **Commit hash** for experimental builds: `abc123def456`
- **Latest** tag only for non-prod environments (use specific tags for prod)

```yaml
# ✅ Good for prod (specific version)
image: 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:v1.0.12

# ⚠️ Avoid for prod (floating tag)
image: 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:latest

# ✅ OK for dev (latest is acceptable)
image: 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog:latest
```

---

### 7. Plan Rollbacks Before Deploying

```bash
# If deployment fails, revert quickly:
git revert <commit-hash>
git push origin main
# Workflow automatically redeploys with reverted changes

# Or manually rollback:
kubectl rollout undo deployment/blog-backend -n dwp-dev
```

**Why**: Minimizes downtime

---

### 8. Document Your Changes

In commit messages and PR descriptions:

```markdown
## What changed?
Updated blog-backend image from v1.0.11 to v1.0.12

## Why?
- Performance improvement: 50% faster API response
- Security fix: patched vulnerability CVE-2025-xxxxx
- New feature: added caching layer

## Where?
- apps/blog/dev/backend/blog-deployment.yaml
- apps/blog/qa/backend/blog-deployment.yaml
- apps/blog/stage/backend/blog-deployment.yaml
- apps/blog/prod/backend/blog-deployment.yaml

## Testing
- ✅ Tested in dev environment
- ✅ Load tested with 1000 req/sec
- ✅ Backward compatible with old clients

## Rollback Plan
If issues detected, revert commit and re-push to main
```

**Why**: Helps team understand context

---

## 🔒 Security Considerations

### 1. Protect Main Branch

**Enable branch protection**:
1. Go to Settings → Branches → Add rule
2. Branch name pattern: `main`
3. Enable:
   - ✅ Require pull request reviews (minimum 1)
   - ✅ Require status checks to pass
   - ✅ Require branches to be up to date
   - ✅ Require code owner reviews
   - ✅ Dismiss stale pull request approvals

**Effect**: Prevents accidental direct pushes to main

---

### 2. Use OIDC for AWS Credentials

❌ **Never** use long-lived AWS access keys in GitHub Actions
✅ **Always** use OIDC (OpenID Connect)

Current workflow uses OIDC:
```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v5
  with:
    role-to-assume: ${{ secrets.AWS_SSM_ROLE }}
```

**Benefits**:
- No static credentials in repository
- Automatic credential rotation
- Time-limited tokens
- Full audit trail

---

### 3. Limit IAM Permissions

GitHub Actions role should have minimum permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand",
        "ssm:ListCommandInvocations",
        "ssm:GetCommandInvocation"
      ],
      "Resource": "*"  // Consider limiting to specific bastion
    }
  ]
}
```

---

### 4. Audit Deployments

**All deployments are logged**:
- GitHub Actions workflow runs
- AWS CloudTrail (SSM commands)
- EKS audit logs (kubectl commands)

**Check logs**:
```bash
# View CloudTrail logs
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=SendCommand \
  --max-results 10
```

---

### 5. Secrets Management

⚠️ **NEVER** commit secrets to Git

Current issues:
- Database passwords hardcoded in manifests
- AWS account ID exposed in image URIs

**Recommended solutions**:
- Use AWS Secrets Manager
- Use Kubernetes Secrets with external-secrets operator
- Use sealed-secrets

---

## 🚀 Future Improvements: ArgoCD Adoption Roadmap

### Vision

Currently, this project implements a **custom GitOps solution** using GitHub Actions + AWS SSM. While this provides excellent control and auditability, migrating to **ArgoCD** in the future will provide:

- **Continuous Reconciliation**: Cluster state automatically synced to Git (no more drift)
- **Native Kubernetes Integration**: GitOps operator running in-cluster
- **Advanced Deployment Strategies**: Blue-green, canary, progressive rollouts
- **Multi-Cluster Management**: Single pane of glass for all environments
- **Reduced Operational Overhead**: No manual polling or SSM orchestration
- **Industry Standard**: Widely adopted in Kubernetes ecosystem

### Why ArgoCD Makes Sense

**Current System Strengths**:
- ✅ Excellent security (OIDC, no long-lived credentials)
- ✅ Full audit trail via AWS services
- ✅ Tight AWS integration
- ✅ Fine-grained control via custom logic

**Current System Limitations**:
- ❌ Manual polling (GitHub Actions waits for SSM status)
- ❌ Push-based model (not pull-based reconciliation)
- ❌ No continuous drift detection
- ❌ Additional operational burden on bastion host
- ❌ Limited to single deployment model
- ❌ Scaling to multiple clusters requires duplication

**ArgoCD Advantages**:
- ✅ Pull-based model (cluster pulls from Git)
- ✅ Continuous reconciliation (drift detection)
- ✅ Native Kubernetes operator
- ✅ Multi-cluster support
- ✅ Web UI for visibility
- ✅ Built-in health tracking
- ✅ Advanced deployment strategies
- ✅ Webhook support for notifications
- ✅ RBAC integrated with Kubernetes

### Migration Strategy

#### **Phase 1: Preparation (Weeks 1-2)**

**Goal**: Design ArgoCD architecture without disrupting current deployments

**Tasks**:

1. **Assess Current State**
   ```bash
   # Count manifests
   find apps/blog -name "*.yaml" | wc -l
   
   # Identify manifest patterns
   find apps/blog -type f -name "*deployment.yaml" | head -5
   ```

2. **Plan ArgoCD Architecture**
   ```
   argocd-system/                    # ArgoCD namespace
   ├── argocd-server                 # API & UI
   ├── argocd-repo-server            # Git repo cloning
   ├── argocd-controller-manager     # Reconciliation
   └── argocd-dex-server             # SSO (optional)
   
   apps/
   ├── blog/
   │   ├── dev/
   │   │   └── argocd-app.yaml       # ArgoCD Application CRD
   │   ├── qa/
   │   │   └── argocd-app.yaml
   │   ├── stage/
   │   │   └── argocd-app.yaml
   │   └── prod/
   │       └── argocd-app.yaml
   ```

3. **Design Repository Structure for ArgoCD**
   ```
   Current Structure (Push-based):
   apps/blog/dev/
   ├── backend/
   ├── frontend/
   ├── namespace/
   └── networking/
   
   ArgoCD-ready Structure (Pull-based):
   apps/blog/dev/
   ├── kustomization.yaml            # Kustomize base
   ├── base/
   │   ├── backend/
   │   ├── frontend/
   │   ├── namespace/
   │   └── networking/
   ├── overlays/
   │   ├── dev/kustomization.yaml    # Env-specific patches
   │   ├── qa/kustomization.yaml
   │   ├── stage/kustomization.yaml
   │   └── prod/kustomization.yaml
   └── argocd-application.yaml       # ArgoCD Application resource
   ```

4. **Create ArgoCD Installation Plan**
   ```bash
   # Add ArgoCD namespace
   kubectl create namespace argocd
   
   # Install ArgoCD (using Helm or manifests)
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   helm install argocd argo/argo-cd \
     -n argocd \
     -f values.yaml
   ```

5. **Plan Cutover Strategy**
   - Keep current system operational
   - Deploy ArgoCD alongside current system
   - Gradually migrate applications one by one
   - Run both in parallel during transition period
   - Final cutover after validation

---

#### **Phase 2: ArgoCD Installation & Configuration (Weeks 3-4)**

**Goal**: Deploy ArgoCD to clusters and configure initial settings

**Tasks**:

1. **Install ArgoCD in Dev Cluster**
   ```bash
   # Create argocd namespace
   kubectl create namespace argocd --context dev-cluster
   
   # Deploy ArgoCD
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --context dev-cluster
   
   # Wait for rollout
   kubectl rollout status deployment argocd-server -n argocd --context dev-cluster
   ```

2. **Configure Git Repository Access**
   ```bash
   # Store GitHub token as Kubernetes secret
   kubectl create secret generic github-creds \
     --from-literal=username=your-username \
     --from-literal=password=your-github-token \
     -n argocd --context dev-cluster
   ```

3. **Configure RBAC in ArgoCD**
   ```yaml
   # argocd-rbac-cm.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: argocd-rbac-cm
     namespace: argocd
   data:
     policy.default: 'role:readonly'
     policy.csv: |
       p, role:dev-admin, applications, *, apps/blog/dev/*, allow
       p, role:qa-admin, applications, *, apps/blog/qa/*, allow
       p, role:stage-admin, applications, *, apps/blog/stage/*, allow
       p, role:prod-admin, applications, *, apps/blog/prod/*, allow
       g, github:devops-team, role:prod-admin
       g, github:dev-team, role:dev-admin
   ```

4. **Setup ArgoCD Web UI Access**
   ```bash
   # Port-forward to access UI
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   
   # Get initial password
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
   
   # Login and change password
   # Access https://localhost:8080
   ```

5. **Configure ArgoCD Projects**
   ```yaml
   # argocd-project.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: blog-app
     namespace: argocd
   spec:
     description: Blog application project
     sourceRepos:
     - 'https://github.com/YOUR_ORG/k8-blogApp.git'
     destinations:
     - namespace: 'dwp-*'
       server: https://kubernetes.default.svc
     clusterResourceWhitelist:
     - group: ''
       kind: Namespace
     namespaceResourceBlacklist:
     - group: ''
       kind: ResourceQuota
   ```

---

#### **Phase 3: Refactor Manifests for Kustomize (Weeks 5-6)**

**Goal**: Reduce duplication using Kustomize overlays

**Current State**: 48+ duplicated YAML files across 4 environments

**New State**: 12 base files + 4 overlay patches

**Tasks**:

1. **Create Base Structure**
   ```
   apps/blog/base/
   ├── kustomization.yaml
   ├── namespace.yaml
   ├── backend/
   │   ├── kustomization.yaml
   │   ├── blog-deployment.yaml
   │   └── blog-service.yaml
   ├── frontend/
   │   ├── kustomization.yaml
   │   ├── blog-ui-deployment.yaml
   │   └── blog-ui-service.yaml
   └── networking/
       ├── kustomization.yaml
       └── frontend-to-backend.yaml
   ```

2. **Create Kustomize Base File**
   ```yaml
   # apps/blog/base/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   
   namespace: dwp-dev
   
   resources:
   - namespace.yaml
   - backend/
   - frontend/
   - networking/
   
   images:
   - name: blog-backend
     newName: 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/blog
     newTag: v1.0.11
   - name: blog-frontend
     newName: 850756192530.dkr.ecr.us-east-1.amazonaws.com/dwp/np/ui
     newTag: 1.1.7
   
   commonLabels:
     app.kubernetes.io/name: blog
     app.kubernetes.io/managed-by: argocd
   
   replicas:
   - name: blog-backend
     count: 2
   - name: blog-frontend
     count: 2
   
   commonAnnotations:
     argocd.argoproj.io/sync-wave: "1"
   ```

3. **Create Environment Overlays**
   ```yaml
   # apps/blog/overlays/dev/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   
   bases:
   - ../../base
   
   namespace: dwp-dev
   
   patchesStrategicMerge:
   - deployment-patches.yaml
   
   images:
   - name: blog-backend
     newTag: v1.0.11  # Dev can use different tag
   - name: blog-frontend
     newTag: 1.1.7
   
   namePrefix: dev-
   
   replicas:
   - name: blog-backend
     count: 2
   - name: blog-frontend
     count: 2
   ```

4. **Create Overlay Patches**
   ```yaml
   # apps/blog/overlays/prod/deployment-patches.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: blog-backend
   spec:
     replicas: 3  # More replicas in prod
     template:
       spec:
         containers:
         - name: blog-backend
           resources:
             requests:
               cpu: 500m
               memory: 512Mi
             limits:
               cpu: 1000m
               memory: 1024Mi
   ```

5. **Validate Kustomize Output**
   ```bash
   # Test build
   kustomize build apps/blog/overlays/dev
   kustomize build apps/blog/overlays/prod
   
   # Verify no errors
   ```

---

#### **Phase 4: Create ArgoCD Applications (Weeks 7-8)**

**Goal**: Define ArgoCD Application resources for each environment

**Tasks**:

1. **Create ArgoCD Application for Dev**
   ```yaml
   # apps/blog/dev/argocd-application.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: blog-app-dev
     namespace: argocd
   spec:
     project: blog-app
     
     source:
       repoURL: https://github.com/YOUR_ORG/k8-blogApp.git
       targetRevision: develop  # Dev tracks develop branch
       path: apps/blog/overlays/dev
     
     destination:
       server: https://kubernetes.default.svc
       namespace: dwp-dev
     
     syncPolicy:
       automated:
         prune: true       # Delete resources removed from Git
         selfHeal: true    # Sync if cluster state diverges
       syncOptions:
       - CreateNamespace=true
     
     health:
       enabled: true
     
     # Notifications
     notifications:
       - destination: slack
         when: sync-failed
     
     # Rollback config
     revision:
       rollbackOnFailure: true
   ```

2. **Create Applications for Other Environments**
   ```yaml
   # apps/blog/qa/argocd-application.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: blog-app-qa
     namespace: argocd
   spec:
     # ... similar to dev but:
     source:
       targetRevision: main
       path: apps/blog/overlays/qa
     # ... rest of config
   
   # apps/blog/stage/argocd-application.yaml
   # apps/blog/prod/argocd-application.yaml
   # (similar with different targetRevisions and syncPolicies)
   ```

3. **Create Application Controller Configuration**
   ```yaml
   # argocd-cm.yaml - ArgoCD ConfigMap
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: argocd-cm
     namespace: argocd
   data:
     application.instanceLabelKey: argocd.argoproj.io/instance
     server.disable.auth: 'false'
     server.insecure: 'false'
     
     # Webhook for GitHub
     application.webhook.github.secret: your-webhook-secret
     
     # Status badge
     application.status.badge.enabled: 'true'
     
     # Health assessment
     application.health.ignore.missing.crd: 'true'
   ```

4. **Setup Webhook for GitHub**
   ```bash
   # Get ArgoCD server URL
   kubectl get service argocd-server -n argocd
   
   # Add webhook to GitHub:
   # Settings → Webhooks → Add webhook
   # Payload URL: https://argocd-server.example.com/api/webhook
   # Content type: application/json
   # Events: Push events + Pull request events
   ```

---

#### **Phase 5: Parallel Operation & Validation (Weeks 9-10)**

**Goal**: Run both systems in parallel to validate ArgoCD before cutover

**Tasks**:

1. **Deploy Blog App via ArgoCD**
   ```bash
   # Apply ArgoCD Application resources
   kubectl apply -f apps/blog/*/argocd-application.yaml -n argocd
   
   # Monitor sync status
   argocd app list
   argocd app get blog-app-dev
   ```

2. **Disable GitHub Actions for Dev**
   ```yaml
   # .github/workflows/deploy.yaml
   on:
     push:
       paths:
       - 'apps/blog/qa/**'      # Disable dev
       - 'apps/blog/stage/**'
       - 'apps/blog/prod/**'
   ```

3. **Monitor ArgoCD Sync Status**
   ```bash
   # Watch application status
   kubectl get applications -n argocd -w
   
   # Check sync history
   argocd app history blog-app-dev
   
   # Monitor pod status
   kubectl get pods -n dwp-dev -w
   ```

4. **Validate Health Checks**
   - Compare ArgoCD health with manual checks
   - Verify pod readiness detection
   - Test deployment rollback via ArgoCD
   - Verify drift detection

5. **Run Parallel Tests**
   ```bash
   # Test case 1: Manual manifest update
   git push # updates manifest
   # Watch ArgoCD sync automatically
   
   # Test case 2: Manual cluster change (drift detection)
   kubectl scale deployment blog-backend -n dwp-dev --replicas=5
   # ArgoCD should detect and correct within minutes
   
   # Test case 3: Rollback
   argocd app rollback blog-app-dev 1
   # Verify pods rollback automatically
   ```

---

#### **Phase 6: Gradual Cutover (Weeks 11-12)**

**Goal**: Migrate all environments from GitHub Actions to ArgoCD

**Steps**:

1. **Dev Environment** (already done in Phase 5)

2. **QA Environment**
   ```bash
   # Update workflow to skip QA
   # Deploy QA via ArgoCD
   kubectl apply -f apps/blog/qa/argocd-application.yaml -n argocd
   # Monitor sync
   # Disable in GitHub Actions
   ```

3. **Stage Environment**
   ```bash
   # Repeat QA steps for stage
   ```

4. **Production Environment** (careful cutover)
   ```bash
   # Requires special approval
   # Setup manual sync policy (no auto-sync for prod)
   # Require approval for syncs
   ```

5. **Decommission GitHub Actions SSM Flow**
   - Remove SSM document dependency
   - Remove bastion host deployment role
   - Remove GitHub Actions OIDC role
   - Document for audit trail

---

### Post-ArgoCD Benefits

#### **1. Continuous Reconciliation**

**Before (Current)**:
```
git push → GitHub Actions workflow → SSM → Bastion → kubectl apply
Time: 40-100 seconds
Drift detection: Manual checks only
```

**After (ArgoCD)**:
```
git push → ArgoCD detects change → Automatic sync
Time: 2-10 seconds
Drift detection: Continuous (every 3 minutes)
```

#### **2. Advanced Deployment Strategies**

```yaml
# Canary Deployment Example
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: blog-app-canary
spec:
  # ... standard config ...
  strategy:
    canary:
      initialSteps:
      - analysis:
          templates:
          - name: cpu-usage
            interval: 1m
            count: 2
      setWeight: 20  # 20% traffic to new version
      analysis:
        interval: 1m
        threshold: 1
        maxWeight: 100
        stepWeight: 20
```

#### **3. Multi-Cluster Management**

```yaml
# Deploy to multiple clusters from single ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: blog-app-global
spec:
  destinations:
  - server: https://dev-cluster.example.com
    namespace: dwp-dev
  - server: https://prod-cluster.example.com
    namespace: dwp-prod
  - server: https://dr-cluster.example.com
    namespace: dwp-dr
```

#### **4. Built-in Notifications**

```yaml
# Slack notifications on sync
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-sync-succeeded: |
    message: |
      Application {{.app.metadata.name}} sync succeeded
      Revision: {{.app.status.operationState.syncResult.revision}}
```

#### **5. Audit & Compliance**

```bash
# Full audit trail
kubectl logs -n argocd -f deployment/argocd-controller-manager

# Track who synced what
argocd app history <app-name>

# RBAC per environment
argocd account get-user-info
```

---

### Timeline & Resource Requirements

| Phase | Duration | Team | Effort |
|-------|----------|------|--------|
| 1. Preparation | 2 weeks | 1 DevOps | 20 hours |
| 2. Installation | 2 weeks | 1-2 DevOps | 30 hours |
| 3. Refactor Manifests | 2 weeks | 1 DevOps | 25 hours |
| 4. Create Applications | 2 weeks | 1-2 DevOps | 30 hours |
| 5. Parallel Operation | 2 weeks | 2 DevOps | 40 hours |
| 6. Cutover | 2 weeks | 2 DevOps | 35 hours |
| **Total** | **12 weeks** | **1-2 FTE** | **180 hours** |

---

### Implementation Checklist

**Prerequisites**:
- [ ] ArgoCD architecture designed
- [ ] Team trained on ArgoCD concepts
- [ ] Kustomize structure planned
- [ ] Webhook infrastructure ready

**Installation**:
- [ ] ArgoCD installed in all clusters
- [ ] Git repository configured
- [ ] RBAC policies defined
- [ ] Notifications configured

**Migration**:
- [ ] Manifests refactored to Kustomize
- [ ] Base and overlay structure created
- [ ] ArgoCD Applications created
- [ ] Parallel operation validated

**Cutover**:
- [ ] Dev environment migrated
- [ ] QA environment migrated
- [ ] Stage environment migrated
- [ ] Prod environment migrated
- [ ] GitHub Actions workflows disabled
- [ ] SSM documents decommissioned

**Post-Migration**:
- [ ] Documentation updated
- [ ] Team trained on ArgoCD operations
- [ ] Monitoring/alerting configured
- [ ] Runbooks created
- [ ] Disaster recovery tested

---

### Backward Compatibility

**Important**: The migration to ArgoCD will be **non-breaking**:

1. **Current manifests remain compatible**
   - No changes to YAML structure
   - All existing deployments continue working
   - Can run both systems in parallel

2. **Gradual migration path**
   - Migrate one environment at a time
   - Keep GitHub Actions as backup
   - Easy rollback if issues occur

3. **Git repository unchanged**
   - Same `apps/blog/{env}` structure
   - Just add ArgoCD Application CRDs
   - No need to restructure existing files

4. **No downtime required**
   - Both systems can coexist
   - Applications deployed by both (idempotent)
   - Gradual cutover over 2-3 weeks

---

### Success Metrics Post-ArgoCD

| Metric | Current | Target |
|--------|---------|--------|
| Deployment latency | 40-100s | 2-10s |
| Time to recover from drift | Manual | <5 minutes |
| Deployment frequency capability | Hourly | Per-minute |
| Rollback time | ~2 minutes | <30 seconds |
| Operational overhead | High (SSM polling) | Low (automatic) |
| Multi-cluster support | Manual | Native |
| Cost per deployment | Medium | Low |

---

### Risk Mitigation

**Risk**: ArgoCD goes down
- **Mitigation**: Kubernetes runs independently; no impact on running pods

**Risk**: Git repo unavailable
- **Mitigation**: Keep cluster state; ArgoCD retries automatically

**Risk**: Network latency affects sync
- **Mitigation**: Configure reasonable sync intervals (3-5 min for prod)

**Risk**: Team unfamiliar with ArgoCD
- **Mitigation**: 4-week training period during Phase 1-4

**Risk**: Manifest refactoring breaks deployments
- **Mitigation**: Extensive testing in dev/qa before prod cutover

---

### Decision Points

**Before Phase 1 - Executive Sign-off**:
- [ ] Budget approved for 12-week project
- [ ] Team capacity allocated
- [ ] Business case accepted

**After Phase 2 - Technical Review**:
- [ ] Architecture approved by platform team
- [ ] RBAC model validated
- [ ] Security review completed

**After Phase 5 - Go/No-Go Decision**:
- [ ] Dev validation completed
- [ ] Performance acceptable
- [ ] No issues in parallel operation

**After Phase 6 - Production Release**:
- [ ] All environments migrated
- [ ] Old systems fully decommissioned
- [ ] Documentation complete

---

### Reference Architecture Post-ArgoCD

```
GitHub Repository
    ↓ (webhook on push)
ArgoCD Server (in-cluster)
    ├── Monitors Git for changes
    ├── Compares desired state (Git) vs actual state (cluster)
    ├── Syncs automatically on drift
    └── Reports status to GitHub via commit status checks
    
EKS Cluster
    ├── ArgoCD controller watches Applications
    ├── Automatic kubectl apply
    ├── Health monitoring
    └── Drift detection & correction

Multiple Environments
    ├── dev-cluster  ← ArgoCD instance (dev/qa)
    ├── prod-cluster ← ArgoCD instance (stage/prod)
    └── dr-cluster   ← ArgoCD instance (disaster recovery)

Observability
    ├── Prometheus (metrics)
    ├── Grafana (dashboards)
    ├── Slack (notifications)
    └── Audit logs (compliance)
```

---

This roadmap provides a clear path to modernize the GitOps infrastructure while maintaining stability and minimizing risk during the transition.

---

## ❓ FAQ

### Q: How often can I deploy?
**A**: As often as you want. The system handles:
- Multiple deployments per day
- Concurrent changes to different environments
- Sequential deployment to same environment (using concurrency control)

---

### Q: What if I accidentally deploy a bad image to prod?
**A**: 
1. Revert the manifest change: `git revert <commit>`
2. Push to main: `git push origin main`
3. Workflow automatically redeployes with old image
4. Rollback completes in ~1 minute

Alternatively, manually rollback:
```bash
kubectl rollout undo deployment/blog-backend -n dwp-prod
```

---

### Q: Can I deploy to multiple environments simultaneously?
**A**: Yes! If you change multiple environments in one commit, the workflow:
1. Detects all changed environments
2. Deploys to them sequentially (not in parallel)
3. Reports status for each

Example:
```bash
# This triggers 3 deployments
sed -i 's/v1.0.11/v1.0.12/g' apps/blog/{dev,qa,stage}/backend/blog-deployment.yaml
git add apps/blog/
git commit -m "Update to v1.0.12"
git push
# Deploys to dev, then qa, then stage
```

---

### Q: What happens if the bastion host is down?
**A**: 
- SSM command fails with error
- Workflow exits with failure code
- GitHub Actions shows ❌ in Actions tab
- On-call engineer is notified

**Recovery**:
1. Restart bastion host
2. Rerun workflow from GitHub Actions UI

---

### Q: Can I schedule deployments?
**A**: Current system is event-driven (push-triggered). To schedule:

Option 1 - Scheduled commits:
```bash
# Create GitHub Actions scheduled workflow
# This can push to main on a schedule
name: Scheduled Deployment
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          git config user.name "bot"
          git commit --allow-empty -m "chore: scheduled deployment"
          git push
```

Option 2 - Manual workflow trigger:
```yaml
on:
  workflow_dispatch:  # Add to deploy.yaml
```

---

### Q: How do I add a new app to this system?
**A**: 
1. Create directory: `apps/myapp/{dev,qa,stage,prod}/`
2. Add manifests: `backend/`, `frontend/`, `namespace/`, `networking/`
3. Update workflow validation to allow your app name:
   ```bash
   if [[ "$appName" != "blog" && "$appName" != "myapp" ]]; then
     # Error
   fi
   ```

---

### Q: How do I add a new environment (e.g., preprod)?
**A**:
1. Create directory: `apps/blog/preprod/`
2. Copy manifests from prod and customize
3. Update workflow to recognize `preprod`:
   ```bash
   if [[ "$envName" != "dev" && "$envName" != "qa" && "$envName" != "stage" && "$envName" != "prod" && "$envName" != "preprod" ]]; then
     # Error
   fi
   ```

---

### Q: How long does deployment take?
**A**: 
- Typical deployment: **40-60 seconds**
- Breakdown:
  - Change detection: ~2 seconds
  - SSM command send: ~1 second
  - Bastion execution: ~20-40 seconds
  - Status polling: ~5-20 seconds
  - Completion: ~1 second

**Factors affecting time**:
- Pod startup time (depends on image size, startup code)
- Resource availability on nodes
- Network latency to ECR

---

### Q: How do I know if deployment succeeded?
**A**: Check in order of preference:

1. **GitHub Actions UI** (fastest)
   - Green checkmark ✅ = Success
   - Red X ❌ = Failure

2. **Kubernetes cluster** (most accurate)
   ```bash
   kubectl rollout status deployment/blog-backend -n dwp-dev
   kubectl get pods -n dwp-dev
   ```

3. **AWS SSM** (audit trail)
   ```bash
   aws ssm list-command-invocations --region us-east-1
   ```

---

### Q: What if I need to rollback?
**A**: 
```bash
# Option 1: Revert commit and push
git log --oneline -5  # Find bad commit
git revert <commit-hash>
git push origin main

# Option 2: Manually rollback in cluster
kubectl rollout undo deployment/blog-backend -n dwp-dev
kubectl rollout status deployment/blog-backend -n dwp-dev

# Option 3: Reapply old manifest
git show <good-commit>:apps/blog/dev/backend/blog-deployment.yaml | kubectl apply -f -
```

---

## 📞 Support & Contribution

**For questions or issues**:
- Create a GitHub Issue with `[GitOps]` prefix
- Tag: `@devops-team`

**Contributing**:
- Follow the patterns in this README
- Test changes in dev first
- Create PR with detailed description
- Get approval from DevOps team lead

---

## 📜 License

See [LICENSE](LICENSE) file for details.

---

**Last Updated**: January 17, 2026  
**Maintained By**: DevOps Team  
**Status**: Production Ready
