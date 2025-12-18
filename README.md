# Amazon EKS at Scale

A comprehensive guide to deploying and managing applications on Amazon Elastic Kubernetes Service (EKS), from cluster provisioning to GitOps-driven deployments.

## Overview

This repository accompanies a five-part article series covering:

| Part | Topic | Description |
|------|-------|-------------|
| 1 | [Introduction to EKS](docs/articles/part-1-introduction-to-eks.md) | EKS fundamentals and managed Kubernetes on AWS |
| 2 | [EKS Architecture](docs/articles/part-2-eks-architecture.md) | Control plane, networking, and IAM integration |
| 3 | [Provisioning with Terraform](docs/articles/part-3-provisioning-eks-with-terraform.md) | Infrastructure as Code for EKS clusters |
| 4 | [Deploying Applications](docs/articles/part-4-deploying-applications-on-eks.md) | Kubernetes Deployments, Services, and Helm |
| 5 | [GitOps with Argo CD](docs/articles/part-5-gitops-with-argocd.md) | Automated deployments using GitOps principles |


## Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Terraform >= 1.0
- kubectl
- Helm 3

## Quick Start

```bash
# Clone the repository
git clone https://github.com/bansikah22/eks-at-scale.git
cd eks-at-scale

# Provision EKS cluster
cd eks-terraform
terraform init
terraform apply

# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

Noel Bansikah - [GitHub](https://github.com/bansikah22)
