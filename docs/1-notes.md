# Amazon Elastic Kubernetes Service (EKS) — Official Docs Summary

## 1. What is Amazon EKS?

Amazon Elastic Kubernetes Service (EKS) is a fully managed Kubernetes service that makes it easy to run, manage, and scale Kubernetes clusters on AWS or in your own data centers. It removes much of the complexity in operating Kubernetes, allowing teams to focus on applications instead of infrastructure.

**Key Benefits**

With EKS you can:
- Deploy apps faster with less operational overhead
- Scale with changing workload demands
- Integrate securely with AWS services
- Choose between different modes of operation
- Run Kubernetes on AWS cloud, on-premises, or hybrid environments (EKS Anywhere & Hybrid Nodes)

## 2. EKS Deployment Modes

### Standard EKS

- AWS manages the Kubernetes control plane (API servers, etcd, schedulers, etc.)
- You manage the worker nodes (compute resources) in your AWS account
- Best choice if you want control over worker nodes, autoscaling, and networking configurations.

### EKS Auto Mode

- AWS also manages Nodes (data plane) on your behalf
- Automates infrastructure provisioning, scaling, cost optimization, OS patching, and security services
- Simplifies cluster operations even more than standard EKS.

## 3. Capabilities of Amazon EKS

EKS provides managed cluster capabilities and tools that extend Kubernetes functionality:

- **Argo CD** — GitOps-based continuous deployment
- **AWS Controllers for Kubernetes (ACK)** — Manage AWS resources from Kubernetes
- **Kube Resource Orchestrator (kro)** — Custom resource orchestration

These features help reduce installation burdens and let you focus on building applications.

## 4. Features of Amazon EKS

### Management Interfaces

You can provision and manage clusters through:
- AWS Console
- CLI tools (eksctl, AWS CLI)
- IaC tools (CloudFormation, Terraform, AWS CDK)
- SDKs/APIs

### Access Control

- Integrates Kubernetes RBAC with AWS IAM
- Supports Service Account IAM Roles for secure workload access to AWS services

### Compute Options

- Supports EC2 nodes (Linux & Windows)
- Use ARM-based Graviton or Nitro instances for performance optimization

### Storage

- EKS Auto Mode can auto-create default storage classes (EBS)
- Supports CSI drivers for S3, EFS, FSx, and other storage options

### Monitoring & Security

- Shared responsibility model for cluster security
- Integrated with CloudWatch, CloudTrail, and observability tools like Prometheus & ADOT Operator

### Kubernetes Compatibility

- EKS is Kubernetes-certified, so community tooling and apps work natively
- Offers standard and extended support for Kubernetes versions

## 5. Integration with AWS Services

You can use many AWS services with your EKS clusters:

| Service                    | Use Case                        |
|----------------------------|---------------------------------|
| EC2                        | Compute (worker nodes)          |
| ECR                        | Container image registry        |
| EBS                        | Block storage                   |
| CloudWatch                 | Monitoring & logs               |
| Elastic Load Balancing     | Traffic distribution            |
| GuardDuty                  | Threat detection                |
| Amazon Managed Prometheus  | Metrics tracking                |
| AWS Resilience Hub         | Cluster resiliency evaluation   |

## 6. Pricing Overview

- You are charged a per-cluster EKS fee
- Compute (EC2 or Fargate), storage, and networking are billed separately
- Savings Plans can be used for cost optimization

Detailed pricing is available on the Amazon EKS pricing pages.