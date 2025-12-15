# Amazon EKS Deployment Options (Official AWS Docs Summary)

Amazon EKS offers multiple ways to deploy Kubernetes clusters, ranging from fully managed cloud clusters to hybrid and on-premises setups. Each option lets you run Kubernetes workloads where it makes the most sense for your applications and operational needs.

## 1. Overview

Amazon EKS is a fully managed Kubernetes service that automates the provisioning and operation of control plane infrastructure, letting you focus on your workloads rather than cluster management. EKS helps ensure scalability, reliability, and security across different deployment environments.

## 2. EKS in the Cloud

This is the most common deployment option where clusters run on AWS infrastructure:

### AWS Regions

- The Kubernetes control plane is fully managed by AWS.
- Worker nodes and compute can be provisioned using:
  - EKS Auto Mode — automates compute, networking, and storage infrastructure.
  - Managed Node Groups — AWS manages EC2 nodes for you.
  - Self-Managed EC2 Nodes — you manage EC2 instances yourself.
  - AWS Fargate — serverless compute that removes the need to manage nodes.
- You get access to the full range of AWS services and integration with AWS networking, security, and observability.

### AWS Local Zones & Wavelength Zones

- EKS can also run compute in Local Zones and Wavelength Zones, which are AWS’s edge infrastructure offerings.
- The control plane still runs in an AWS Region, while compute runs close to end users for low-latency workloads.
- Available compute models here include Managed Node Groups and self-managed EC2 nodes (Fargate and Auto Mode may not be available depending on zone features).

## 3. EKS in Your Data Center or at the Edge

If you need to run workloads outside the AWS cloud:

### AWS Outposts

- A fully managed AWS service that extends AWS infrastructure and services into your own facilities or co-location sites.
- EKS clusters here can run compute on EC2 instances managed by AWS Outposts.
- The Kubernetes control plane stays in an AWS Region, while your worker nodes can be local.
- You can also deploy the entire cluster locally on AWS Outposts using EKS local clusters (including the control plane).

### EKS Hybrid Nodes

- Use your own physical or virtual servers in data centers or edge environments as Kubernetes worker nodes.
- The EKS control plane remains AWS-managed in a Region.
- This lets you run container workloads in places where cloud connectivity or latency constraints matter.

| Deployment         | Control Plane  | Worker Nodes        | Location         |
|--------------------|---------------|---------------------|------------------|
| AWS Outposts       | AWS-managed   | EC2 self-managed    | Local / On-prem  |
| EKS Hybrid Nodes   | AWS-managed   | Customer-managed    | On-prem / Edge   |

## 4. EKS Anywhere (Air-Gapped / Customer-Managed)

Amazon EKS Anywhere allows you to create and operate self-managed Kubernetes clusters outside AWS cloud infrastructure:

- Your organization is fully responsible for cluster lifecycle operations, updates, and maintenance.
- It’s built on Kubernetes Cluster API (CAPI) for declarative cluster management using familiar tooling.
- Supports environments like VMware vSphere, bare metal, Nutanix, Apache CloudStack, and AWS Snow.
- Runs even in air-gapped environments with optional integration to AWS identity and observability services.
- Enterprise support is available via EKS Anywhere Enterprise Subscription for add-ons and support.

| Feature         | EKS Anywhere      |
|-----------------|------------------|
| Control Plane   | Customer-managed |
| Worker Nodes    | Customer-managed |
| Location        | On-prem / Edge / Air-gapped |

## 5. EKS Tooling That Supports Deployment

AWS provides tooling to help interact with and manage EKS clusters across deployment options:

- **Amazon EKS Connector** – Register and view any conformant Kubernetes cluster (including those not hosted on AWS) in the EKS console. It provides visibility — but not lifecycle management — from the AWS console.
- **Amazon EKS Distro (EKS-D)** – The same Kubernetes distribution used by EKS in the cloud. You can use EKS-D to build and manage your own Kubernetes clusters using your preferred tooling (not covered by standard AWS Support Plans).

## Quick Comparison of Options

| Option                    | Managed Control Plane | Managed Workers   | Good For                       |
|---------------------------|----------------------|-------------------|--------------------------------|
| AWS Regions (Standard EKS)| Yes                  | Yes / Partial     | Cloud-native production apps   |
| AWS Local/Wavelength Zones| Yes                  | Partial           | Low-latency edge workloads     |
| Outposts                  | Yes                  | Partial           | On-prem enterprise apps        |
| EKS Hybrid Nodes          | Yes                  | No                | Customer-owned infra           |
| EKS Anywhere              | No                   | No                | Air-gapped or self-managed clusters |

## Summary

Amazon EKS allows you to deploy Kubernetes clusters:
- Fully managed in the AWS cloud, with optional automation via Auto Mode or Fargate.
- At the edge or on-premises using AWS Outposts or hybrid worker nodes tied back to the AWS control plane.
- Completely self-managed (control plane + workers) with Amazon EKS Anywhere for flexibility in disconnected or custom environments.