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
![alt](https://github.com/nawaz-ch/CI-CD/blob/c16b262fff51e9669b963d9c9bc271b351c7cf20/21_01_01_GitOps_CI.png)
