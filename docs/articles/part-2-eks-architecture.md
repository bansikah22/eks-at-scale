# Amazon EKS Series - Part 2: EKS Architecture and Core Components

---
title: Amazon EKS Series - Part 2: EKS Architecture and Core Components
published: true
description: Deep dive into Amazon EKS architecture - understand the control plane, worker nodes, networking, and IAM integration.
tags: aws, kubernetes, eks, devops
series: Amazon EKS at Scale
cover_image: 
---

## Introduction

Welcome back to the **Amazon EKS at Scale** series!

In [Part 1](./part-1-introduction-to-eks.md), we covered the fundamentals of Amazon EKS — what it is, why to use it, and the different ways to manage worker nodes. We also created our first EKS cluster using `eksctl`.

In this article, we'll take a deeper look at:
- The high-level architecture of Amazon EKS
- Control plane components (AWS-managed)
- Worker nodes (customer-managed)
- Networking fundamentals
- IAM and authentication

Understanding this architecture is essential before diving into production deployments.

---

## High-Level View of Amazon EKS

Amazon EKS is a **managed Kubernetes service** that runs the Kubernetes control plane for you. This means AWS handles the complex, undifferentiated heavy lifting of running Kubernetes, while you focus on deploying and managing your applications.

![EKS Architecture](/docs/images/kubernetes-in-action.png)

### What AWS Manages vs What You Manage

| Component | Managed By |
|-----------|------------|
| Control Plane (API Server, etcd, Scheduler, Controllers) | AWS |
| Control Plane High Availability | AWS |
| Control Plane Security Patches | AWS |
| Worker Nodes | You (or AWS with Managed Node Groups/Fargate) |
| Application Deployments | You |
| Pod Networking | You (with AWS VPC CNI) |
| IAM Roles and Policies | You |

This shared responsibility model allows you to leverage AWS's expertise in running highly available infrastructure while maintaining control over your workloads.

---

## EKS Control Plane (AWS-Managed)

The control plane is the brain of your Kubernetes cluster. In EKS, AWS fully manages this component, running it across **multiple Availability Zones** for high availability.

### Core Control Plane Components

#### 1. Kubernetes API Server (`kube-apiserver`)

The API server is the **front door** to your Kubernetes cluster:

- Exposes the Kubernetes API over HTTPS
- Validates and processes all API requests (from `kubectl`, controllers, and other components)
- Acts as the gateway for all cluster operations — creating pods, services, deployments, etc.
- Authenticates requests using AWS IAM (via the AWS IAM Authenticator)

When you run `kubectl get pods`, your request goes to the API server, which retrieves the information from etcd and returns it to you.

#### 2. etcd

etcd is a **distributed key-value store** that serves as Kubernetes' database:

- Stores all cluster state and configuration data
- Holds information about pods, services, secrets, ConfigMaps, and more
- Provides strong consistency guarantees
- In EKS, AWS manages etcd replication across multiple Availability Zones

You never interact with etcd directly — all access goes through the API server.

#### 3. Scheduler (`kube-scheduler`)

The scheduler is responsible for **placing pods on nodes**:

- Watches for newly created pods that have no node assigned
- Evaluates resource requirements (CPU, memory, storage)
- Considers constraints like node selectors, taints, tolerations, and affinity rules
- Selects the most suitable node and binds the pod to it

The scheduler ensures efficient resource utilization across your cluster.

#### 4. Controller Manager (`kube-controller-manager`)

The controller manager runs **control loops** that regulate cluster state:

- **Node Controller** — Monitors node health and responds when nodes go down
- **Replication Controller** — Ensures the correct number of pod replicas are running
- **Endpoints Controller** — Populates endpoint objects (joins Services and Pods)
- **Service Account Controller** — Creates default service accounts for new namespaces

Each controller watches the current state and works to move it toward the desired state.

### Key Characteristics of EKS Control Plane

- **Runs in an AWS-managed VPC** — Isolated from your account and other customers
- **Highly available** — At least two API server instances and three etcd nodes across multiple AZs
- **Automatically scaled** — AWS scales control plane resources based on cluster size
- **No direct access** — You cannot SSH into control plane nodes; you interact only via the API
- **Automatic updates** — AWS handles patching and security updates

---

## Worker Nodes (Customer-Managed)

Worker nodes are the **compute resources** where your applications actually run. Unlike the control plane, you are responsible for provisioning and managing worker nodes (unless using Fargate).

### Worker Node Components

Each worker node runs several Kubernetes components:

#### 1. kubelet

The kubelet is the **primary node agent**:

- Registers the node with the Kubernetes API server
- Watches for pods scheduled to its node
- Ensures containers are running and healthy
- Reports node and pod status back to the control plane
- Executes liveness and readiness probes

The kubelet communicates with the container runtime to manage container lifecycle.

#### 2. kube-proxy

kube-proxy handles **networking on each node**:

- Maintains network rules for pod-to-pod communication
- Implements Kubernetes Services (ClusterIP, NodePort, LoadBalancer)
- Uses iptables or IPVS to route traffic to the correct pods
- Enables service discovery within the cluster

#### 3. Container Runtime

The container runtime **executes containers**:

- EKS uses `containerd` as the default runtime (Docker support was deprecated in Kubernetes 1.24)
- Pulls container images from registries (like Amazon ECR)
- Creates and manages container processes
- Handles container isolation using Linux namespaces and cgroups

### Worker Node Options in EKS

| Option | Node Management | Scaling | Best For |
|--------|----------------|---------|----------|
| Self-Managed Nodes | You manage everything | Manual or custom | Full control, custom AMIs |
| Managed Node Groups | AWS manages provisioning and updates | Auto Scaling Groups | Most production workloads |
| AWS Fargate | AWS manages everything | Automatic per-pod | Serverless, variable workloads |

---

## Networking in EKS

Networking is a critical aspect of any Kubernetes deployment. EKS integrates deeply with AWS networking services.

### VPC (Virtual Private Cloud)

Your EKS cluster runs inside a **VPC** — an isolated virtual network in AWS:

- Provides network isolation and security
- You define the IP address range (CIDR block)
- Contains subnets, route tables, and internet gateways
- EKS requires a VPC with subnets in at least two Availability Zones

### Subnets

Subnets divide your VPC into smaller network segments:

#### Public Subnets
- Have a route to an Internet Gateway
- Resources can have public IP addresses
- Used for load balancers and bastion hosts
- NAT Gateways are placed here for private subnet internet access

#### Private Subnets
- No direct route to the internet
- Access the internet via NAT Gateway (for pulling images, etc.)
- **Recommended for worker nodes** — provides better security
- Pods and nodes are not directly accessible from the internet

### AWS VPC CNI (Container Network Interface)

EKS uses the **Amazon VPC CNI plugin** for pod networking:

- Each pod gets a **real IP address** from your VPC CIDR range
- Pods can communicate directly with other AWS services (RDS, ElastiCache, etc.)
- No overlay network — native VPC networking performance
- Supports security groups for pods (with specific configurations)

**How it works:**
1. The CNI plugin attaches Elastic Network Interfaces (ENIs) to worker nodes
2. Each ENI can have multiple secondary IP addresses
3. These IPs are assigned to pods running on the node
4. Pod-to-pod traffic uses native VPC routing

**Important consideration:** The number of pods per node is limited by the number of ENIs and IPs the instance type supports.

### Simplified Network Flow

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Cloud                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                      Your VPC                          │  │
│  │  ┌─────────────────┐    ┌─────────────────┐           │  │
│  │  │  Public Subnet  │    │  Public Subnet  │           │  │
│  │  │   (AZ-1)        │    │   (AZ-2)        │           │  │
│  │  │  Load Balancer  │    │  NAT Gateway    │           │  │
│  │  └─────────────────┘    └─────────────────┘           │  │
│  │  ┌─────────────────┐    ┌─────────────────┐           │  │
│  │  │ Private Subnet  │    │ Private Subnet  │           │  │
│  │  │   (AZ-1)        │    │   (AZ-2)        │           │  │
│  │  │  Worker Nodes   │    │  Worker Nodes   │           │  │
│  │  │  (Pods)         │    │  (Pods)         │           │  │
│  │  └─────────────────┘    └─────────────────┘           │  │
│  └───────────────────────────────────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────┴───────────────────────────┐  │
│  │           EKS Control Plane (AWS-Managed)             │  │
│  │     API Server  │  etcd  │  Scheduler  │  Controllers │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## IAM and Authentication

EKS uses **AWS IAM** for authentication, providing a secure and familiar way to manage access to your cluster.

### How Authentication Works

1. **User runs kubectl command** — e.g., `kubectl get pods`
2. **AWS IAM Authenticator** — kubectl uses the AWS CLI to get a token
3. **Token sent to API Server** — The token is included in the request
4. **EKS validates the token** — Confirms the IAM identity
5. **Kubernetes RBAC** — Determines what the user can do

### IAM vs Kubernetes RBAC

These are two separate but complementary systems:

| Aspect | AWS IAM | Kubernetes RBAC |
|--------|---------|-----------------|
| **Purpose** | *Who* can access the cluster | *What* they can do inside |
| **Scope** | AWS account level | Kubernetes cluster level |
| **Managed by** | AWS IAM policies | Kubernetes Role/ClusterRole |
| **Example** | "User X can call EKS APIs" | "User X can list pods in namespace Y" |

### The aws-auth ConfigMap

The `aws-auth` ConfigMap maps IAM identities to Kubernetes users and groups:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/admin
      username: admin
      groups:
        - system:masters
```

**Key points:**
- Worker nodes use IAM roles mapped in `aws-auth` to join the cluster
- You can map IAM users and roles to Kubernetes groups
- The `system:masters` group has full cluster admin access
- EKS also supports EKS access entries (a newer, simpler alternative to aws-auth)

### Best Practices for IAM in EKS

- **Use IAM roles, not users** — Roles are more secure and support temporary credentials
- **Follow least privilege** — Grant only the permissions needed
- **Use IRSA (IAM Roles for Service Accounts)** — Allow pods to assume IAM roles for AWS API access
- **Audit access regularly** — Review who has access to your cluster

---

## Summary

In this article, we explored the architecture of Amazon EKS:

- **EKS is a managed Kubernetes service** — AWS handles the control plane, you manage worker nodes and applications
- **Control plane components** — API Server, etcd, Scheduler, and Controller Manager work together to manage cluster state
- **Worker nodes** — Run kubelet, kube-proxy, and container runtime to execute your workloads
- **Networking** — EKS uses VPC networking with the AWS VPC CNI plugin, giving pods real VPC IP addresses
- **IAM integration** — Authentication uses AWS IAM, while authorization uses Kubernetes RBAC

---

## Key Takeaways

- The EKS control plane runs in an AWS-managed VPC, highly available across multiple AZs
- You never access control plane nodes directly — only through the Kubernetes API
- Worker nodes run in your VPC and are your responsibility (unless using Fargate)
- Pods get real VPC IP addresses, enabling direct communication with AWS services
- IAM handles *authentication* (who you are), RBAC handles *authorization* (what you can do)
- The `aws-auth` ConfigMap bridges IAM identities to Kubernetes users

---

## What's Next?

In **Part 3**, we'll get hands-on and provision an Amazon EKS cluster using **Terraform** and community modules. You'll learn:
- Setting up Terraform for EKS
- Using the terraform-aws-eks module
- Configuring VPC, subnets, and node groups
- Best practices for infrastructure as code

Stay tuned!

---

## Resources

- [Amazon EKS Architecture - AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Amazon VPC CNI Plugin](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [Cluster Authentication - AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

---

*Found this article helpful? Follow along with the series and share your thoughts in the comments!*
