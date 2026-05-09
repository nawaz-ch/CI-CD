# DevOps CI/CD with GitHub Actions & ArgoCD on AWS EKS
implement Production-Grade CI/CD Pipelines for our real-world retail app on AWS EKS using:
- GitHub Actions for CI (Build & Push to ECR)
- ArgoCD for CD (Deploy to EKS using Helm)
- Amazon ECR for image storage
- OIDC-based IAM roles (no hardcoded AWS secrets)

use the ui microservice from our real-world retail app and walk through the complete process — from code changes to EKS deployment.

# What to build
| Demo | Concepts |
|------|--------|
|01_CI_github_actions_AWS_ECR | 	GitHub Actions workflow to build Docker image and push to Amazon ECR using secure OIDC |
|02_CD_ArgoCD_Install | 	ArgoCD installation on EKS, access via UI/CLI, login, and admin password handling |
| 03_CD_ArgoCD_Helm | ArgoCD + Helm integration, GitOps-style sync policies, auto-deploy with values.yaml |
| 04_CI_CD_Full_Flow_Test| 	End-to-end flow: UI version update → GitHub Actions CI → ArgoCD CD → EKS deployment |

# Tools & Technologies
- GitHub Actions (CI)
- AWS ECR (container registry)
- AWS EKS (Kubernetes cluster)
- ArgoCD (CD controller)
- Helm (package manager for Kubernetes)
- OIDC IAM Role (secure GitHub → AWS access)

#  Flow Summary
 - You push a code change to the ui microservice
 - GitHub Actions builds Docker image → tags → pushes to ECR
 - Helm values-ui.yaml is updated with new image tag
 - ArgoCD syncs from Git → deploys new version to EKS cluster

![alt](https://github.com/nawaz-ch/CI-CD/blob/c16b262fff51e9669b963d9c9bc271b351c7cf20/02_github_actions.png)
![alt](https://github.com/nawaz-ch/CI-CD/blob/c16b262fff51e9669b963d9c9bc271b351c7cf20/argocd-retail-ui.png)

# GitHub Actions CI: Build & Push UI Microservice to AWS ECR
This guide establishes a secure, GitOps-ready CI pipeline using GitHub Actions with OIDC authentication (no AWS keys stored in GitHub!). The pipeline automatically builds, tags, and pushes Docker images to Amazon ECR, then updates Helm values to trigger ArgoCD deployments.

# What This Pipeline Achieves
```bash
Code Change -> GitHub Actions -> Build Image -> Push to ECR -> Update Helm Values -> ArgoCD Deploys
```
**Key Features:**
- Secure OIDC authentication (no long-lived AWS credentials)
- Dual tagging strategy (latest + sha-commit)
- GitOps-ready (auto-updates Helm values)
- Fast, automated CI triggered on code changes

# GitOps Full Flow (DevOps CI and CD combined flow)
```bash
Developer Pushes Code to src/ui/src/**
            |
            | (1) Triggers GitHub Actions workflow
            v
    GitHub Actions Runner
            |
            | (2) Builds Docker image
            | (3) Pushes to ECR with tags: latest + sha-a1b2c3d
            | (4) Updates chart/values-ui.yaml (tag: sha-a1b2c3d)
            | (5) Commits and pushes to Git
            v
       Git Repository (main branch)
            |
            | (6) ArgoCD polls Git every 3 minutes
            v
      ArgoCD Detects Change
            |
            | (7) Syncs and deploys new image
            v
        EKS Cluster (Running Pods)
```
**CI Pipeline Flow**
![alt](https://github.com/nawaz-ch/CI-CD/blob/077fdebfa3c4624fd64c5a94148e216664c3e72c/21_01_01_GitOps_CI.png)

![alt](https://github.com/nawaz-ch/CI-CD/blob/c16b262fff51e9669b963d9c9bc271b351c7cf20/01_github_actions.png)

![alt](https://github.com/nawaz-ch/CI-CD/blob/c16b262fff51e9669b963d9c9bc271b351c7cf20/02_github_actions.png)

![alt](https://github.com/nawaz-ch/CI-CD/blob/fe9e45cf5e748cb31341f6e0cee732b4d0bdd2fd/21_01_04_ecr_image.png)

![alt](https://github.com/nawaz-ch/CI-CD/blob/e94ec9cc76007dc61c1d0aa75dc3a595d7b231e4/21_01_05_values_ui_image_tag.png)

# Create ECR Repository
```bash
# Create ECR repository for UI microservice
aws ecr create-repository \
  --repository-name retail-store/ui \
  --region us-east-1

# Expected output:
# {
#     "repository": {
#         "repositoryArn": "arn:aws:ecr:us-east-1:123456789012:repository/retail-store/ui",
#         "repositoryName": "retail-store/ui",
#         "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/retail-store/ui"
#     }
# }
```
# Create GitHub OIDC IAM Role
**Set Environment Variables**
```bash
# Set your configuration
AWS_REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
GITHUB_REPO="nawaz-ch/aws-devops-github-actions-ecr-argocd3"  # UPDATE with YOUR repo
ROLE_NAME="github-actions-oidc-role-ui3"

# Verify variables are set correctly
echo "AWS Region: $AWS_REGION"
echo "Account ID: $ACCOUNT_ID"
echo "GitHub Repo: $GITHUB_REPO"
echo "IAM Role Name: $ROLE_NAME"
```
**Generate Trust Policy**
```bash
# Generate trust-policy.json with automatic variable substitution
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GITHUB_REPO}:*"
        }
      }
    }
  ]
}
EOF

# Verify the generated file
echo "Trust policy created. Contents:"
cat trust-policy.json
```
What this does: Allows GitHub Actions from your repository to assume this IAM role using OIDC tokens (no AWS keys needed!)

** Create IAM Role**
```bash
# Verify the trust policy before creating role
cat trust-policy.json | jq '.'

# Create the IAM role
aws iam create-role \
  --role-name $ROLE_NAME \
  --assume-role-policy-document file://trust-policy.json

# Expected output:
# {
#     "Role": {
#         "RoleName": "github-actions-oidc-role-ui3",
#         "Arn": "arn:aws:iam::123456789012:role/github-actions-oidc-role-ui3",
#         ...
#     }
# }
```
**Attach ECR Permissions**
```bash
# Attach AWS managed policy for ECR push/pull access
aws iam attach-role-policy \
  --role-name $ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

# Verify policy is attached
aws iam list-attached-role-policies --role-name $ROLE_NAME
```
**What this grants:**
-  Push images to ECR
-   Pull images from ECR
-   Manage ECR repositories
-   Get ECR authentication tokens

**Create the OIDC Provider in Your AWS Account**
```bash
# List OIDC Providers
aws iam list-open-id-connect-providers

# Create OIDC Provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com 

# List OIDC Providers
aws iam list-open-id-connect-providers
```

# Configure GitHub Actions Workflow
**Create GitHub Repository and Copy Files**
- Repo Name: aws-devops-github-actions-ecr-argocd2
- COPY FILES: COPY ALL FILES FROM **github-files** directory

**Update Workflow File**
- Edit `.github/workflows/build-push-ui.yaml` and update the role ARN:
```bash
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-oidc-role-ui3  # Replace <ACCOUNT_ID>
    aws-region: ${{ env.AWS_REGION }}

```

**Verify Workflow Configuration**
Key workflow settings to verify:
```bash
env:
  AWS_REGION: us-east-1                    # Your region
  ECR_REPOSITORY: retail-store/ui          # Your ECR repo name

permissions:
  id-token: write   # Required for OIDC
  contents: write   # Required to commit Helm updates

```

**Push Configuration to GitHub**
```bash
# Navigate to your repository
cd aws-devops-github-actions-ecr-argocd3

# Stage all changes
git add .

# Commit with descriptive message
git commit -m "Add GitHub Actions CI workflow for UI microservice"

# Push to main branch (triggers the workflow!)
git push origin main
```
> Note: This initial push will NOT trigger the workflow because no changes were made to src/ui/src/**

# Understanding the Workflow
**what this workflow does**
| Step | Action | Description |
|------|----------|-------------|
| 1. Checkout code | `actions/checkout@v4` | Clones repository code to GitHub Actions runner |
| 2. Configure AWS credentials | `configure-aws-credentials@v4` | 	Uses OIDC to assume IAM role (keyless auth) |
|3. Login to Amazon ECR | `	amazon-ecr-login@v2` | Authenticates Docker with ECR using temporary credentials |
| 4. Define image tags | Shell script |Generates latest and sha-<commit> tags (e.g., sha-a1b2c3d) |
|5. Build and push images | `docker build + docker push` |Builds image once, pushes both tags to ECR |
|6. Setup Git auth| 	Git config |	Configures ci-bot identity with GITHUB_TOKEN |
|7. Update Helm values | `sed + git commit/push` | Updates chart/values-ui.yaml with SHA tag, pushes to Git |
|8. CI Complete | Log output | 	Confirms successful completion |


** sha-<commit> Tag (Example: sha-a1b2c3d)**
- Purpose: Immutable reference tied to Git commit
- Behavior: Never changes (permanent)
- Used in: chart/values-ui.yaml (production deployments)
**Example in Helm values file:**
```bash
image:
  repository: 180789647333.dkr.ecr.us-east-1.amazonaws.com/retail-store/ui
  tag: sha-a1b2c3d  # SHA tag only (immutable)
  # NOT using "latest" - too risky for production!
```

# GitOps Flow Explained
```bash
Developer Pushes Code to src/ui/src/**
            |
            | (1) Triggers GitHub Actions workflow
            v
    GitHub Actions Runner
            |
            | (2) Builds Docker image
            | (3) Pushes to ECR with tags: latest + sha-a1b2c3d
            | (4) Updates chart/values-ui.yaml (tag: sha-a1b2c3d)
            | (5) Commits and pushes to Git
            v
       Git Repository (main branch)
            |
            | (6) ArgoCD polls Git every 3 minutes
            v
      ArgoCD Detects Change
            |
            | (7) Syncs and deploys new image
            v
        EKS Cluster (Running Pods)

```
> Key Principle: Git is the single source of truth for both code and deployment state!

# Test the CI Pipeline
**Make a Code Change**
update the UI version to trigger the workflow:
```bash
# Sync your local repo (always pull latest first!)
git pull origin main
or
./git-pull.sh

# Option 1: Use provided script
./update-ui-home-html.sh V102

# Option 2: Manual edit
# Edit src/ui/src/home.html and change the version:
```

**Find this line:**
```bash
<h1 class="text-4xl sm:text-5xl font-bold text-white mb-6">
  The most public <span class="text-primary-400">Secret Shop - Version: V101</span>
</h1>
```
**changes to**
```bash
<h1 class="text-4xl sm:text-5xl font-bold text-white mb-6">
  The most public <span class="text-primary-400">Secret Shop - Version: V102</span>
</h1>
```

## Commit and Push
```bash
# Stage changes
git add src/ui/src/home.html

# Commit with version in message
git commit -m "Update UI to Version V102"

# Push to trigger workflow
git push origin main

or
./git-push.sh
```

# Monitor GitHub Actions Workflow
**Watch the Pipeline**
- Go to your GitHub repository
- Click Actions tab
- You should see a new workflow run: "Build and Push UI Service to ECR"

# Verify Successful Completion
**Look for these success indicators in the logs:**
```bash
[PASS] Checkout code                        
[PASS] Configure AWS credentials via OIDC   
[PASS] Login to Amazon ECR                  
[PASS] Define image tags                    
[PASS] Build and push Docker images         
[PASS] Setup Git auth using GITHUB_TOKEN    
[PASS] Update Helm values file              
[PASS] CI Complete                          
```

# ArgoCD Installation on EKS (Production-Ready)
**What is Argo CD?**
![alt](https://github.com/nawaz-ch/CI-CD/blob/c16b262fff51e9669b963d9c9bc271b351c7cf20/21_02_01_ArgoCD.png)

**Create Namespace for ArgoCD**
```bash
kubectl create namespace argocd
```

**Install ArgoCD Core Components**
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs:
- ArgoCD UI Server
- Repository Server
- Application Controller
- All ArgoCD CRDs

**Access ArgoCD UI Locally (Port Forward)**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
```bash
https://localhost:8080
```

**Get ArgoCD Admin Password**
```bash
# Get ArgoCD Admin Password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode && echo
```
Use username admin and the above password to log in to the web UI.

**Login via argocd CLI (Optional but Recommended)**
```bash
argocd login localhost:8080 --username admin --password <copied-password> --insecure
```
**Change the Admin Password**
```bash
argocd account update-password
```

# ArgoCD Deployment for Retail UI (Helm-based GitOps)
**deploying the retail-ui microservice using ArgoCD + Helm in a GitOps-driven CI/CD workflow.**

# Introduction

