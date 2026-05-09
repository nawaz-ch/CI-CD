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

![alt]()
