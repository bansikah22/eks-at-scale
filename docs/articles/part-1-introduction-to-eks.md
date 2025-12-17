# Amazon EKS Series - Part 1: Introduction to Amazon Elastic Kubernetes Service

---
title: Amazon EKS Series - Part 1: Introduction to EKS
published: true
description: Learn the fundamentals of Amazon Elastic Kubernetes Service (EKS) - what it is, why use it, and the different ways to manage worker nodes.
tags: aws, kubernetes, eks, devops
series: Amazon EKS at Scale
cover_image: 
---

## Introduction

Welcome to the first part of the **Amazon EKS at Scale** series! In this series, we'll explore Amazon Elastic Kubernetes Service (EKS) from the ground up, covering everything from basic concepts to advanced cluster management.

![Kubernetes Architecture](/docs/images/kubernetes-in-action.png)

In this article, we'll cover:
- What is Amazon EKS?
- Why use EKS?
- Understanding Worker Node options
- Creating and connecting to an EKS cluster

---

## What is Amazon EKS?

Amazon Elastic Kubernetes Service (EKS) is a **managed Kubernetes service** provided by AWS. It simplifies running Kubernetes by handling the complex parts of cluster management for you.

### What AWS Manages for You

With EKS, AWS takes care of:

- **Provisioning and maintaining master nodes** — AWS automatically sets up and maintains the master nodes that run the Kubernetes control plane, eliminating the need for you to manage this critical infrastructure.

- **Installing control plane processes:**
  - **API Server** — The front-end for the Kubernetes control plane. It handles all REST requests for modifications to pods, services, and other resources.
  - **Scheduler** — Watches for newly created pods with no assigned node and selects a node for them to run on based on resource requirements.
  - **etcd** — A consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data.

- **Scaling and backups of the cluster** — AWS automatically handles scaling the control plane and maintains backups of your cluster state, ensuring high availability.

- **Patching and updating control plane components** — Security patches and version updates are managed by AWS, keeping your control plane secure and up-to-date.

This means you can focus on deploying your applications rather than managing Kubernetes infrastructure.

---

## Why Use EKS?

There are several compelling reasons to choose EKS:

1. **Simplified Operations** — Running and scaling Kubernetes on your own requires significant expertise. You need to manage etcd clusters, configure API servers, and handle upgrades carefully. EKS removes this complexity by providing a production-ready control plane out of the box.

2. **Enhanced Security** — Kubernetes security involves network policies, RBAC, secrets management, and more. EKS integrates natively with AWS IAM for authentication, making it easier to implement least-privilege access. Your control plane runs in an AWS-managed VPC, isolated from other customers.

3. **AWS Integration** — EKS works seamlessly with services like:
   - **IAM** for authentication and authorization
   - **VPC** for network isolation
   - **CloudWatch** for logging and monitoring
   - **ELB** for load balancing
   - **ECR** for container image storage

   This tight integration simplifies building production-grade applications.

---

## Worker Nodes: Your Options

While EKS manages the control plane, **you are responsible for the worker nodes** (where your applications actually run). There are three ways to set up worker nodes:

### 1. Self-Managed Nodes

With self-managed nodes, you have complete control but also full responsibility:

- **Manual EC2 provisioning** — You select and launch EC2 instances yourself, choosing instance types, AMIs, and configurations.
- **Install Kubernetes components** — All worker processes must be installed manually:
  - `kubelet` — The primary node agent that ensures containers are running in pods
  - `kube-proxy` — Maintains network rules for pod communication
  - **Container runtime** — Software that runs containers (e.g., containerd, Docker)
- **Ongoing maintenance** — Security patches, OS updates, and Kubernetes version upgrades are your responsibility.
- **Node registration** — You must configure nodes to register with the EKS control plane.

**Best for:** Teams needing full control over node configuration, custom AMIs, or specific compliance requirements.

### 2. Managed Node Groups

Managed node groups let AWS handle the heavy lifting while you retain some control:

- **Automated provisioning** — AWS creates and configures EC2 instances for you using EKS-optimized AMIs that come pre-configured with the necessary Kubernetes components.
- **Simplified lifecycle management** — Create, update, or terminate nodes with a single API call. AWS handles rolling updates gracefully.
- **Auto Scaling integration** — Each node group is backed by an Auto Scaling group, allowing automatic scaling based on demand.
- **Managed updates** — AWS can update nodes to new AMI versions while respecting pod disruption budgets.

**Best for:** Most production workloads where you want a balance of control and convenience.

### 3. AWS Fargate

Fargate provides a fully serverless approach to running containers:

- **No node management** — You don't provision, configure, or scale EC2 instances. AWS handles all infrastructure.
- **On-demand compute** — Fargate spins up compute resources only when pods are scheduled, and removes them when pods terminate.
- **Right-sized resources** — Based on your pod's CPU and memory requirements, Fargate automatically provisions appropriately sized compute.
- **Pay-per-use pricing** — You only pay for the vCPU and memory your pods actually consume, billed per second.

**Best for:** Variable or unpredictable workloads, cost optimization, and teams wanting zero infrastructure management.

---

## Creating an EKS Cluster

### Prerequisites

To create an EKS cluster, you need to configure several components:

1. **Cluster configuration:**
   - **Cluster name** — A unique identifier for your cluster within the region
   - **Kubernetes version** — The version of Kubernetes to run (EKS supports multiple versions)

2. **IAM role for the cluster** — An IAM role that grants EKS permissions to:
   - **Provision nodes** — Create and manage EC2 instances for worker nodes
   - **Access storage** — Interact with EBS volumes and other storage services
   - **Manage secrets** — Access AWS Secrets Manager or Systems Manager Parameter Store

3. **Networking:**
   - **VPC and Subnets** — The network where your cluster will run. Subnets should span multiple Availability Zones for high availability.
   - **Security groups** — Firewall rules controlling traffic to and from your cluster components

### Creating Worker Nodes

After creating the cluster, set up your node group:

1. Create a Node Group (a group of nodes for your Kubernetes environment)
2. Select instance type
3. Define min/max number of nodes
4. Specify which EKS cluster to connect to

---

## Methods to Create an EKS Cluster

There are three primary ways to create an EKS cluster, each with its own trade-offs:

### 1. AWS Console

The web-based approach using the AWS Management Console:

- **Visual interface** — Step-by-step wizards guide you through cluster creation
- **Good for learning** — Helps you understand all the components involved
- **More time-consuming** — Requires manually configuring each component (VPC, subnets, IAM roles, etc.)
- **Not easily reproducible** — Manual steps can lead to inconsistencies between environments

**Best for:** First-time users learning EKS or one-off cluster setups.

### 2. eksctl CLI

A purpose-built command-line tool that simplifies EKS cluster creation:

```bash
eksctl create cluster
```

This single command automatically provisions:
- **VPC and networking** — Creates a new VPC with public and private subnets across multiple AZs
- **Security groups** — Configures appropriate firewall rules for cluster communication
- **IAM roles** — Creates necessary roles for the cluster and node groups
- **Node groups** — Launches worker nodes with EKS-optimized AMIs
- **kubectl configuration** — Updates your local kubeconfig for immediate cluster access

**Best for:** Quick cluster creation, development environments, and teams wanting simplicity.

### 3. Infrastructure as Code (IaC)

Using declarative tools to define your cluster configuration:

- **Terraform** — HashiCorp's popular IaC tool with excellent AWS support
- **Pulumi** — Allows defining infrastructure using familiar programming languages
- **AWS CloudFormation** — AWS's native IaC service using YAML/JSON templates

**Benefits of IaC:**
- **Version control** — Track infrastructure changes in Git
- **Reproducibility** — Create identical clusters across environments
- **Review process** — Infrastructure changes can go through code review
- **Documentation** — Your infrastructure code serves as documentation

**Best for:** Production environments, teams practicing GitOps, and organizations requiring audit trails.

---

## Hands-On Demo: Creating Your First EKS Cluster

### Step 1: Install eksctl

Follow the [official AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) to install eksctl.

### Step 2: Ensure AWS Authentication

Make sure your AWS credentials are configured properly.

### Step 3: Explore eksctl Commands

```bash
# Get help
eksctl --help

# Explore create options
eksctl create --help

# Explore cluster creation options
eksctl create cluster --help
```

### Step 4: Create the Cluster

```bash
eksctl create cluster \
  -n cluster1 \
  --nodegroup-name ng1 \
  --region us-east-1 \
  --node-type t2.micro \
  --nodes 2
```

Wait for the provisioning to complete.

![Provisioning Complete](/docs/images/provision-complete-terminal.png)

### Step 5: Verify Created Resources

After creation, check the following in AWS Console:
- VPC
- Subnets
- EKS Services
- Compute > Node Group
- EC2 Instances

![Cluster with Version](/docs/images/cluster-aws-with-version.png)

---

## Connecting to Your EKS Cluster

### Verify kubectl Configuration

```bash
kubectl config view
```

### Check Your Nodes

```bash
kubectl get nodes
```

![kubectl get nodes](/docs/images/kubectl-get-nodes.png)

If you see your worker nodes listed, your connection is successful!

---

## Cleaning Up

To avoid unnecessary charges, delete your cluster when done:

```bash
eksctl delete cluster -n cluster1
```

![Delete Cluster](/docs/images/eksctl-delete-cluster.png)

---

## Summary

In this first part of the series, we covered:

- **What EKS is** - A managed Kubernetes service where AWS handles the control plane
- **Why use EKS** - Simplified operations, enhanced security, and AWS integration
- **Worker node options** - Self-managed, Managed Node Groups, and Fargate
- **Cluster creation methods** - Console, eksctl, and IaC
- **Basic cluster operations** - Creating, connecting, and deleting clusters

---

## What's Next?

In **Part 2**, we'll dive deeper into:
- EKS architecture and components
- Networking in EKS
- Security best practices

Stay tuned!

---

## Resources

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/)
- [eksctl Official Documentation](https://eksctl.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

*Did you find this article helpful? Follow along with the series and drop a comment with your questions!*
