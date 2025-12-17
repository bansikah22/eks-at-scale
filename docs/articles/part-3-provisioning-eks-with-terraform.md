# Amazon EKS Series - Part 3: Provisioning an EKS Cluster with Terraform

---
title: Amazon EKS Series - Part 3: Provisioning an EKS Cluster with Terraform
published: true
description: Learn how to provision a production-ready Amazon EKS cluster using Terraform and community modules step by step.
tags: aws, kubernetes, terraform, devops
series: Amazon EKS at Scale
cover_image: 
---

## Introduction

Welcome back to the **Amazon EKS at Scale** series!

In [Part 2](./part-2-eks-architecture.md), we explored the architecture of Amazon EKS — the control plane, worker nodes, networking, and IAM integration. Now it's time to put that knowledge into practice.

In this article, you'll learn how to:
- Set up a Terraform project for EKS
- Use community modules for VPC and EKS
- Provision a production-ready EKS cluster
- Verify your cluster with kubectl

**What we're building:** A fully functional EKS cluster with a VPC, private subnets, and managed node groups — all defined as code.

---

## Why Terraform for EKS?

While `eksctl` is great for quick setups (as we saw in Part 1), production environments demand more control and repeatability. That's where **Terraform** shines.

### Benefits of Infrastructure as Code (IaC)

- **Reproducibility** — Create identical clusters across dev, staging, and production environments
- **Version Control** — Track all infrastructure changes in Git with full history
- **Team Collaboration** — Review infrastructure changes through pull requests
- **Automation** — Integrate with CI/CD pipelines for automated deployments
- **Documentation** — Your code serves as living documentation of your infrastructure

### Why Community Modules?

We'll use the official Terraform AWS modules:
- `terraform-aws-modules/vpc/aws`
- `terraform-aws-modules/eks/aws`

These modules are:
- **Production-tested** — Used by thousands of organizations
- **Actively maintained** — Regular updates and security patches
- **Well-documented** — Comprehensive examples and documentation
- **Interview-friendly** — Widely recognized in the industry

---

## Tools & Prerequisites

Before we begin, ensure you have the following installed and configured.

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| AWS CLI | v2.x | AWS authentication |
| Terraform | >= 1.0 | Infrastructure provisioning |
| kubectl | Latest | Kubernetes cluster interaction |

### Installing the Tools

#### AWS CLI

```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli

# Verify installation
aws --version
```

#### Terraform

```bash
# Linux (Ubuntu/Debian)
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify installation
terraform --version
```

#### kubectl

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# macOS
brew install kubectl

# Verify installation
kubectl version --client
```

### AWS Credentials

Configure your AWS credentials using one of these methods:

#### Option 1: AWS Configure (Recommended for beginners)

```bash
aws configure
```

You'll be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Default output format (e.g., `json`)

#### Option 2: Environment Variables

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

#### Verify AWS Configuration

```bash
# Verify AWS CLI is configured
aws sts get-caller-identity
```

You should see your AWS account ID and user/role information:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

### IAM Permissions Required

Your AWS user/role needs permissions to create:
- VPC and networking resources (subnets, NAT gateway, etc.)
- EKS clusters and node groups
- IAM roles and policies
- EC2 instances

For learning purposes, you can use the `AdministratorAccess` policy. For production, follow the principle of least privilege.

---

## Project Structure

A clean project structure makes your Terraform code maintainable. Here's what we'll create:

```
eks-terraform/
├── main.tf          # Main resources (VPC & EKS modules)
├── providers.tf     # Provider configuration
├── versions.tf      # Terraform and provider versions
├── variables.tf     # Input variables
└── outputs.tf       # Output values
```

### Why This Structure?

- **`versions.tf`** — Pin Terraform and provider versions for consistency
- **`providers.tf`** — Configure AWS provider settings
- **`variables.tf`** — Define configurable inputs (region, cluster name, etc.)
- **`main.tf`** — The core infrastructure definitions
- **`outputs.tf`** — Expose useful values after deployment

This separation keeps your code organized and easy to navigate.

---

## Step 1: Terraform Versions

First, let's pin our Terraform and provider versions. This ensures consistent behavior across different machines and CI/CD pipelines.

**`versions.tf`**

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**Why version pinning matters:**
- Prevents unexpected breaking changes
- Ensures team members use compatible versions
- Makes builds reproducible over time

---

## Step 2: Provider Configuration

Configure the AWS provider with the region variable.

**`providers.tf`**

```hcl
provider "aws" {
  region = var.aws_region
}
```

Simple and clean — the region comes from our variables file.

---

## Step 3: Input Variables

Define the configurable parameters for your infrastructure.

**`variables.tf`**

```hcl
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
  default     = "eks-cluster"
}

variable "cluster_version" {
  description = "Kubernetes version for the EKS cluster"
  type        = string
  default     = "1.31"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {
  description = "Environment name (e.g., dev, staging, prod)"
  type        = string
  default     = "dev"
}
```

**Key variables explained:**
- **`aws_region`** — Where your cluster will be deployed
- **`cluster_name`** — Identifier for your EKS cluster
- **`cluster_version`** — Kubernetes version (use a recent stable version)
- **`vpc_cidr`** — IP address range for your VPC (10.0.0.0/16 gives you 65,536 IPs)
- **`environment`** — Useful for tagging and distinguishing environments

---

## Step 4: Main Configuration (VPC & EKS)

This is where the magic happens. We'll use community modules to create our VPC and EKS cluster.

**`main.tf`**

```hcl
# -----------------------------------------------------------------------------
# Data Sources
# -----------------------------------------------------------------------------
data "aws_availability_zones" "available" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

# -----------------------------------------------------------------------------
# Local Variables
# -----------------------------------------------------------------------------
locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)

  tags = {
    Environment = var.environment
    Terraform   = "true"
    Project     = "eks-at-scale"
  }
}

# -----------------------------------------------------------------------------
# VPC Module
# -----------------------------------------------------------------------------
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = var.vpc_cidr

  azs             = local.azs
  private_subnets = [for k, v in local.azs : cidrsubnet(var.vpc_cidr, 4, k)]
  public_subnets  = [for k, v in local.azs : cidrsubnet(var.vpc_cidr, 8, k + 48)]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  # Tags required for EKS
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }

  tags = local.tags
}

# -----------------------------------------------------------------------------
# EKS Module
# -----------------------------------------------------------------------------
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  cluster_endpoint_public_access = true

  # Cluster Add-ons
  cluster_addons = {
    coredns                = {}
    eks-pod-identity-agent = {}
    kube-proxy             = {}
    vpc-cni                = {}
  }

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # EKS Managed Node Group(s)
  eks_managed_node_groups = {
    default = {
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = ["t3.medium"]

      min_size     = 2
      max_size     = 4
      desired_size = 2
    }
  }

  # Cluster access entry
  enable_cluster_creator_admin_permissions = true

  tags = local.tags
}
```

### Breaking Down the Configuration

#### Data Sources

```hcl
data "aws_availability_zones" "available" { ... }
```

Fetches available AZs in the region, filtering out any that require opt-in.

#### Local Variables

```hcl
locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
  ...
}
```

- Selects the first 3 availability zones
- Defines common tags for all resources

#### VPC Module

The VPC module creates:
- **VPC** with the specified CIDR block
- **Public subnets** — For load balancers and NAT gateways
- **Private subnets** — For worker nodes (more secure)
- **NAT Gateway** — Allows private subnets to access the internet
- **Required tags** — EKS uses these tags to discover subnets for load balancers

**Subnet tagging is critical:**
- `kubernetes.io/role/elb = 1` — Public subnets for internet-facing load balancers
- `kubernetes.io/role/internal-elb = 1` — Private subnets for internal load balancers

#### EKS Module

The EKS module creates:
- **EKS Cluster** — The managed control plane
- **Cluster Add-ons** — Essential components (CoreDNS, VPC CNI, kube-proxy)
- **Managed Node Group** — Worker nodes with auto-scaling

**Node group configuration:**
- `ami_type: AL2023_x86_64_STANDARD` — Amazon Linux 2023 (latest EKS-optimized AMI)
- `instance_types: ["t3.medium"]` — 2 vCPUs, 4GB RAM (good for learning)
- `min_size: 2, max_size: 4` — Auto-scaling boundaries
- `desired_size: 2` — Initial number of nodes

> **Note on Instance Types:**  
> For learning and testing purposes, it is possible to use small instance types such as `t2.micro`. However, due to limited CPU and memory, this setup is not recommended for running real workloads. For a more stable experience, instance types like `t3.small` or `t3.medium` are preferred.

---

## Step 5: Outputs

Define useful values to display after deployment.

**`outputs.tf`**

```hcl
output "cluster_name" {
  description = "Name of the EKS cluster"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = module.eks.cluster_endpoint
}

output "cluster_version" {
  description = "Kubernetes version of the cluster"
  value       = module.eks.cluster_version
}

output "cluster_security_group_id" {
  description = "Security group ID attached to the EKS cluster"
  value       = module.eks.cluster_security_group_id
}

output "region" {
  description = "AWS region"
  value       = var.aws_region
}

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "configure_kubectl" {
  description = "Command to configure kubectl"
  value       = "aws eks update-kubeconfig --region ${var.aws_region} --name ${module.eks.cluster_name}"
}
```

The `configure_kubectl` output is especially helpful — it gives you the exact command to run after deployment.

---

## Step 6: Deploy the Infrastructure

Now let's provision our EKS cluster!

### Initialize Terraform

```bash
cd eks-terraform
terraform init
```

This downloads the required providers and modules. You'll see output like:

```
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...

Terraform has been successfully initialized!
```

### Review the Plan

```bash
terraform plan
```

This shows what Terraform will create without making any changes. Review the output carefully — you should see resources for:
- VPC and subnets
- Internet Gateway and NAT Gateway
- Route tables
- EKS cluster
- Managed node group
- Security groups

### Apply the Configuration

```bash
terraform apply
```

Type `yes` when prompted to confirm.

**Note:** This process takes approximately 10-15 minutes. EKS cluster creation is the longest step.

You'll see progress output as resources are created:

```
module.vpc.aws_vpc.this[0]: Creating...
module.vpc.aws_vpc.this[0]: Creation complete after 2s
...
module.eks.aws_eks_cluster.this[0]: Creating...
module.eks.aws_eks_cluster.this[0]: Still creating... [5m0s elapsed]
module.eks.aws_eks_cluster.this[0]: Creation complete after 9m32s
...
Apply complete! Resources: 60 added, 0 changed, 0 destroyed.
```

---

## Step 7: Configure kubectl & Verify

### Update kubeconfig

After successful deployment, configure kubectl to connect to your cluster:

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
```

Or use the output from Terraform:

```bash
$(terraform output -raw configure_kubectl)
```

You should see:

```
Added new context arn:aws:eks:us-east-1:ACCOUNT_ID:cluster/eks-cluster to /home/user/.kube/config
```

### Verify the Cluster

Check that your nodes are ready:

```bash
kubectl get nodes
```

Expected output:

```
NAME                             STATUS   ROLES    AGE   VERSION
ip-10-0-1-xxx.ec2.internal      Ready    <none>   5m    v1.31.x
ip-10-0-2-xxx.ec2.internal      Ready    <none>   5m    v1.31.x
```

Check cluster information:

```bash
kubectl cluster-info
```

Verify system pods are running:

```bash
kubectl get pods -n kube-system
```

You should see CoreDNS, kube-proxy, and VPC CNI pods running.

---

## Step 8: Clean Up (Important!)

To avoid unnecessary AWS charges, destroy the resources when you're done:

```bash
terraform destroy
```

Type `yes` to confirm. This will remove all resources created by Terraform.

**Warning:** This is irreversible and will delete your entire cluster and VPC.

---

## Best Practices for IaC

Here are key best practices to follow as you work with Terraform:

- **Use community modules** — Don't reinvent the wheel; leverage tested modules
- **Pin versions** — Lock Terraform, provider, and module versions
- **Keep state remote** — Use S3 + DynamoDB for team collaboration (we'll cover this later)
- **Separate environments** — Use workspaces or separate directories for dev/staging/prod
- **Use variables** — Make your code reusable across environments
- **Tag everything** — Tags help with cost tracking and resource management
- **Review plans carefully** — Always check `terraform plan` before applying
- **Use .gitignore** — Never commit `.terraform/` or `*.tfstate` files

---

## Summary

In this article, we:

- Set up a clean Terraform project structure
- Used community modules for VPC and EKS
- Provisioned a production-ready EKS cluster
- Configured kubectl and verified the cluster
- Learned IaC best practices

You now have a repeatable, version-controlled way to create EKS clusters!

---

## What's Next?

In **Part 4**, we'll deploy applications to our EKS cluster and explore:
- Deploying a sample application
- Understanding Kubernetes Deployments and Services
- Exposing applications with Load Balancers
- Introduction to Helm for package management

Stay tuned!

---

## Resources

- [Terraform AWS VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)
- [Terraform AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
- [Amazon EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [Terraform Documentation](https://www.terraform.io/docs)

---

## Code Repository

All code from this article is available in the `eks-terraform/` directory of the series repository.

---

*Found this article helpful? Follow along with the series and drop a comment with your questions!*
