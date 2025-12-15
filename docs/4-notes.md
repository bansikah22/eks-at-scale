# Kubernetes Concepts (Amazon EKS Docs — Summarized & Structured)

Amazon EKS is a managed Kubernetes service, but once your cluster is running, you use Kubernetes just like you would on any other platform. To work with an EKS cluster, you need a basic understanding of Kubernetes core concepts.
![Kubernetes in action](/docs/images/kubernetes-in-action.png)

## 1. Why Kubernetes?

Kubernetes is designed to help you run containerized applications reliably and at scale. Its purpose is to automate many of the complex operational tasks that occur when managing distributed applications.

**Kubernetes Provides:**
- Multi-machine deployment: Run and manage containers across many machines.
- Health monitoring: Restart containers that fail.
- Scaling: Automatically scale applications up or down.
- Rolling updates: Deploy new versions of containers without downtime.
- Resource allocation: Share CPU and memory fairly across workloads.
- Load balancing: Distribute traffic across containers.

Developers describe the desired state of their apps using YAML manifests, and Kubernetes constantly works to make the actual state match the desired state.

## 2. Core Kubernetes Cluster Concepts

### What Is a Kubernetes Cluster?

A cluster is a group of machines that run containerized applications. It consists of:
- **Control Plane** – Coordinates the cluster (managed by AWS in EKS).
- **Worker Nodes** – Where your applications actually run.

## 3. Cluster Creation and Management

You can create and manage clusters using tools specific to providers:

**Common Tools:**
- `eksctl` – EKS-specific CLI that simplifies cluster creation.
- Terraform / CloudFormation – Infrastructure-as-code tools for declarative provisioning.
- Console or AWS CLI – AWS’s interfaces for cluster creation.

When creating an EKS cluster, AWS sets up networking, control plane components, and can provision worker nodes such as EC2 instances.

### Control Plane Components

The control plane provides the brain of the cluster and includes:
- **API Server (kube-apiserver):** Handles all Kubernetes API requests.
- **etcd:** Distributed store that persists cluster state.
- **Controller Manager:** Watches cluster state and ensures desired state is maintained.
- **Cloud Controller Manager:** Integrates Kubernetes with cloud provider APIs (e.g., load balancers, network routing).

In EKS, AWS manages this control plane for you, so you don’t need to provision or maintain these components.

### Worker Nodes (Data Plane)

Worker nodes run your workloads and include:
- **kubelet:** Agent that ensures containers are running as defined.
- **Container Runtime (containerd):** Pulls and runs container images.
- **kube-proxy:** Manages networking and service routing on nodes.

Nodes together form the data plane, where containers are scheduled and executed.

## 4. Workloads: How Apps Run on Kubernetes

### Containers and Pods

- A container is the smallest runnable unit (holds runtime + filesystem).
- A Pod groups one or more containers that share networking and storage.
- Pods are described using specifications that declaratively define what should run.

## 5. Common Kubernetes Workload Resources

These objects define how your applications run and scale:

| Resource     | Purpose                                      |
|--------------|----------------------------------------------|
| Deployment   | Manages stateless workloads and replicas     |
| StatefulSet  | Runs stateful applications needing stable identity |
| DaemonSet    | Ensures a Pod runs on every node             |
| Job / CronJob| Runs batch or scheduled tasks                |

## 6. Networking: Services & Ingress

Kubernetes networking allows Pods to communicate:

- **Service:** Stable logical endpoint to access Pods.
- **Ingress:** Rules for routing external traffic into Services.

These help expose apps internally (within the cluster) or externally to users.

## 7. Cluster Extensions & Add-Ons

Kubernetes clusters often use additional components that run inside the cluster (not part of the control plane):

- **CoreDNS:** DNS for service discovery.
- **VPC CNI:** Integrates Pods with AWS networking.
- **Observability tools:** Monitoring, logging, and tracing add-ons.
- **Storage CSI Drivers:** Attach persistent volumes (EBS, EFS, etc.).

Amazon EKS automatically installs some essential add-ons when you create a cluster.

## Summary

- Kubernetes automates container orchestration and is declarative.
- Control plane manages the cluster state and scheduling.
- Worker nodes run application workloads.
- Workloads are described via Pods and higher-level controllers like Deployments.
- Networking is handled with Services and Ingress.
- Add-ons extend cluster capabilities.