

# AWS EKS Pipeline

This repo contains:

* Terraform code to create VPC, EKS cluster, node groups, and ECR
* A sample application with Dockerfile
* A Helm chart for deployment
* A Jenkins pipeline (`Jenkinsfile`) to automate end-to-end CI/CD



## 1. Repository Structure

```
aws-eks-pipeline/
├── app/                # Application + Dockerfile
├── terraform/          # EKS + VPC + ECR infra
├── helm/my-app/        # Helm chart
└── Jenkinsfile         # Full CI/CD automation
```



## 2. Workflow Overview

### Terraform

Creates:

* VPC
* Subnets
* EKS Cluster
* Node Groups
* ECR Repository (Jenkins reads the URL)

### Docker

Builds image from `app/` and pushes to ECR.

### Helm

Deploys the application to EKS using the built image.

### Jenkins Pipeline Stages

1. Checkout
2. Configure AWS Credentials (from Jenkins Credentials Store)**
3. Terraform Init + Apply
4. Docker Build + Push to ECR
5. Helm Upgrade/Install to EKS

A single **Build Now** in Jenkins performs full provisioning + deployment.



## 3. Requirements (on Jenkins server)

* AWS CLI v2
* Terraform ≥ 1.4
* Docker CE
* kubectl ≥ 1.32
* eksctl
* Helm ≥ 3.11



## 4. How to Run

1. Add AWS credentials to Jenkins (ID = `aws-creds`)
2. Create a Jenkins Pipeline job → point to this repo
3. Click **Build Now**
4. Jenkins will:

   * Provision infra using Terraform
   * Build and push Docker image
   * Deploy application to EKS via Helm


## 5. Validation Commands


* kubectl get nodes
* kubectl get pods
* kubectl get svc
* kubectl get ingress



