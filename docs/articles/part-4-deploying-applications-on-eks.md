# Amazon EKS Series - Part 4: Deploying Applications on Amazon EKS

---
title: Amazon EKS Series - Part 4: Deploying Applications on Amazon EKS
published: true
description: Learn how to deploy and expose applications on Amazon EKS using Kubernetes Deployments, Services, and Helm.
tags: aws, kubernetes, eks, devops
series: Amazon EKS at Scale
cover_image: 
---

## Introduction

Welcome back to the **Amazon EKS at Scale** series!

In [Part 3](./part-3-provisioning-eks-with-terraform.md), we provisioned a production-ready EKS cluster using Terraform and community modules. Now it's time to put that cluster to work!

In this article, you'll learn how to:
- Deploy a simple application using Kubernetes
- Understand Deployments and Pods
- Expose applications with Services
- Use AWS Load Balancers with EKS
- Get started with Helm for package management

**Prerequisites:** Ensure your EKS cluster from Part 3 is running and `kubectl` is configured.

```bash
# Verify cluster connection
kubectl get nodes
```

You should see your worker nodes in `Ready` status.

---

## Choosing a Sample Application

When learning Kubernetes, it's best to start with simple applications. This lets you focus on **Kubernetes concepts** rather than application complexity.

### Why Simple Applications?

- **Fewer moving parts** — Easier to debug
- **Quick feedback** — Fast deployment cycles
- **Focus on fundamentals** — Learn the platform, not the app

### Our Choice: NGINX

We'll deploy **NGINX** — a lightweight web server that:
- Starts quickly
- Has a small image size
- Returns a simple webpage (easy to verify)
- Is widely used in examples and documentation

Later, we'll also explore deploying with Helm using a different application.

---

## Understanding Kubernetes Deployments

Before we deploy anything, let's understand what a **Deployment** is.

### What is a Deployment?

A Deployment is a Kubernetes resource that:
- **Declares the desired state** of your application
- **Manages Pods** (the smallest deployable units)
- **Handles updates** with rolling deployments
- **Ensures availability** by maintaining replica counts

Think of it as telling Kubernetes: *"I want 3 copies of this application running at all times."* Kubernetes then makes it happen and keeps it that way.

### What is a Pod?

A **Pod** is the smallest unit in Kubernetes:
- Contains one or more containers
- Shares networking and storage
- Has a unique IP address within the cluster
- Is ephemeral (can be replaced at any time)

**Important:** You rarely create Pods directly. Instead, you create Deployments that manage Pods for you.

### Deployment YAML Structure

Here's what a minimal Deployment looks like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Breaking Down the YAML

| Field | Purpose |
|-------|---------|
| `apiVersion` | API version for this resource type |
| `kind` | Type of resource (Deployment) |
| `metadata.name` | Name of the Deployment |
| `spec.replicas` | Number of Pod copies to run |
| `spec.selector` | How the Deployment finds its Pods |
| `spec.template` | Template for creating Pods |
| `containers` | List of containers in each Pod |
| `image` | Docker image to use |
| `containerPort` | Port the container listens on |

---

## Deploying Our First Application

Let's deploy NGINX to our EKS cluster.

### Step 1: Create the Deployment

Create a file named `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

> **Note:** We've added resource requests and limits. This is a best practice that helps Kubernetes schedule Pods efficiently and prevents any single Pod from consuming too many resources.

### Step 2: Apply the Deployment

```bash
kubectl apply -f nginx-deployment.yaml
```

Output:

```
deployment.apps/nginx-deployment created
```

### Step 3: Verify the Deployment

Check the Deployment status:

```bash
kubectl get deployments
```

Output:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s
```

Check the Pods:

```bash
kubectl get pods
```

Output:

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6d6565499c-2xjkl   1/1     Running   0          45s
nginx-deployment-6d6565499c-8mzft   1/1     Running   0          45s
nginx-deployment-6d6565499c-vwxyz   1/1     Running   0          45s
```

All 3 replicas are running!

### Step 4: View More Details

For detailed information about a Pod:

```bash
kubectl describe pod <pod-name>
```

To see which nodes the Pods are running on:

```bash
kubectl get pods -o wide
```

---

## Understanding Kubernetes Services

Our NGINX Pods are running, but how do we access them?

### The Problem

- Pods have IP addresses, but they're **internal** to the cluster
- Pods are **ephemeral** — they can be replaced at any time
- Each Pod has a **different IP** address

If you try to access a Pod directly by IP, what happens when that Pod dies and a new one takes its place with a different IP?

### The Solution: Services

A **Service** provides:
- **Stable endpoint** — A single IP/DNS name that doesn't change
- **Load balancing** — Distributes traffic across all matching Pods
- **Service discovery** — Other apps can find your service by name

### Service Types

Kubernetes offers several Service types:

| Type | Description | Use Case |
|------|-------------|----------|
| `ClusterIP` | Internal IP only (default) | Internal communication between services |
| `NodePort` | Exposes on each node's IP at a static port | Development, testing |
| `LoadBalancer` | Provisions an external load balancer | Production traffic from internet |

### ClusterIP (Default)

- Only accessible **within** the cluster
- Great for internal services (databases, caches, etc.)
- Other Pods can access via `service-name.namespace.svc.cluster.local`

### NodePort

- Exposes the service on each node's IP at a specific port (30000-32767)
- Accessible from outside the cluster via `<NodeIP>:<NodePort>`
- Not recommended for production (no load balancing across nodes)

### LoadBalancer

- **Best for production workloads on AWS**
- Automatically provisions an AWS Elastic Load Balancer (ELB)
- Provides a public DNS name
- Handles SSL termination (with proper configuration)

---

## Exposing Our Application with LoadBalancer

Let's create a LoadBalancer Service to expose our NGINX deployment to the internet.

### Step 1: Create the Service

Create a file named `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Understanding the Service YAML

| Field | Purpose |
|-------|---------|
| `type: LoadBalancer` | Tells Kubernetes to provision an external load balancer |
| `selector.app: nginx` | Routes traffic to Pods with label `app: nginx` |
| `port: 80` | Port the Service listens on |
| `targetPort: 80` | Port on the Pods to forward traffic to |

### Step 2: Apply the Service

```bash
kubectl apply -f nginx-service.yaml
```

Output:

```
service/nginx-service created
```

### Step 3: Watch the Service Status

```bash
kubectl get service nginx-service --watch
```

Initially, you'll see `<pending>` for the external IP:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-service   LoadBalancer   172.20.10.123   <pending>     80:31234/TCP   10s
```

After 1-2 minutes, AWS provisions the load balancer:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE
nginx-service   LoadBalancer   172.20.10.123   a1b2c3d4e5f6g7h8i9.us-east-1.elb.amazonaws.com   80:31234/TCP   90s
```

Press `Ctrl+C` to stop watching.

### What Happened Behind the Scenes?

When you created the LoadBalancer Service:

1. **Kubernetes** detected the Service type and called the AWS cloud controller
2. **AWS** provisioned a Classic Load Balancer (or NLB with annotations)
3. **The ELB** was configured to forward traffic to your worker nodes
4. **kube-proxy** on each node routes traffic to the correct Pods

This integration is one of the powerful benefits of running Kubernetes on AWS!

---

## Verifying the Application

### Get the Load Balancer URL

```bash
kubectl get service nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Or simply:

```bash
kubectl get svc nginx-service
```

### Access via Browser

Open the EXTERNAL-IP URL in your browser:

```
http://a1b2c3d4e5f6g7h8i9.us-east-1.elb.amazonaws.com
```

You should see the default NGINX welcome page!

### Access via curl

```bash
# Store the LoadBalancer URL in a variable and curl it
LB_URL=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$LB_URL
```

Output:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

### Check Pod Logs

View the access logs from NGINX:

```bash
kubectl logs -l app=nginx --tail=10
```

You'll see HTTP request logs from your browser/curl requests.

---

## Scaling the Application

One of Kubernetes' strengths is easy scaling.

### Scale Up

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

```bash
kubectl get pods
```

You now have 5 NGINX Pods!

### Scale Down

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

Kubernetes will gracefully terminate 3 Pods.

### Autoscaling (Preview)

For automatic scaling based on CPU/memory, you can use the Horizontal Pod Autoscaler (HPA). We'll cover this in a future article.

---

## Introduction to Helm

So far, we've been writing YAML files manually. This works, but what happens when:
- You need to deploy to multiple environments?
- You want to share your deployment with others?
- Applications require dozens of YAML files?

### What is Helm?

**Helm** is the package manager for Kubernetes. Think of it like:
- `apt` for Ubuntu
- `brew` for macOS
- `npm` for Node.js

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Chart** | A package containing all Kubernetes manifests for an application |
| **Release** | A deployed instance of a chart |
| **Repository** | A collection of charts (like a package registry) |
| **Values** | Configuration options to customize a chart |

### Why Use Helm?

- **Templating** — Use variables instead of hardcoded values
- **Reusability** — Package once, deploy many times
- **Versioning** — Track and rollback releases
- **Community** — Thousands of pre-built charts available
- **Simplicity** — Deploy complex apps with one command

### Helm vs Raw YAML

| Aspect | Raw YAML | Helm |
|--------|----------|------|
| Learning curve | Lower | Slightly higher |
| Flexibility | Full control | Template-based |
| Reusability | Copy-paste | Charts and values |
| Complex apps | Many files to manage | Single command |
| Environment config | Manual changes | Values files |

---

## Installing Helm

### Install Helm CLI

```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS
brew install helm

# Verify installation
helm version
```

### Helm Basics

```bash
# Add a chart repository (example with prometheus-community)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update repository information
helm repo update

# Search for charts
helm search repo prometheus
```

> **Note:** Many popular Helm chart repositories like Bitnami have moved to enterprise licensing models. When choosing charts, prefer community-maintained repositories or official project charts. Always check the license and image sources before using a chart in production.

---

## Deploying Apache (Manual Approach)

While Helm is powerful, sometimes the simplest approach is best. Let's deploy Apache using the official Docker Hub image with manual manifests - similar to how we deployed NGINX.

### Why Manual Deployment?

- **No licensing concerns** - Uses official Docker Hub images
- **Full control** - You understand exactly what's deployed
- **Simpler debugging** - Fewer abstractions to troubleshoot
- **Learning value** - Reinforces Kubernetes concepts

### Step 1: Create the Apache Deployment

Create a file named `apache-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  labels:
    app: apache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:2.4
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

The `httpd:2.4` image is the official Apache HTTP Server image from Docker Hub.

### Step 2: Create the Apache Service

Create a file named `apache-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-service
  labels:
    app: apache
spec:
  type: LoadBalancer
  selector:
    app: apache
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Step 3: Deploy Apache

```bash
kubectl apply -f apache-deployment.yaml -f apache-service.yaml
```

Output:

```
deployment.apps/apache-deployment created
service/apache-service created
```

### Step 4: Verify the Deployment

Check the pods:

```bash
kubectl get pods -l app=apache
```

Output:

```
NAME                               READY   STATUS    RESTARTS   AGE
apache-deployment-6c57ff8d-cdfzr   1/1     Running   0          75s
apache-deployment-6c57ff8d-nbdjq   1/1     Running   0          75s
apache-deployment-6c57ff8d-xqv5f   1/1     Running   0          75s
```

### Step 5: Get the LoadBalancer URL

```bash
kubectl get svc apache-service --watch
```

Wait for the EXTERNAL-IP to be assigned (1-2 minutes):

```
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE
apache-service   LoadBalancer   172.20.86.183   a4108c17eb1b441978cb345f8622d540-727057055.us-east-1.elb.amazonaws.com   80:31783/TCP   60s
```

### Step 6: Test the Application

> **Note:** AWS ELB DNS propagation can take 2-5 minutes. If you get a DNS resolution error, wait a moment and try again.

```bash
LB_URL=$(kubectl get svc apache-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$LB_URL
```

Output:

```html
<html><body><h1>It works!</h1></body></html>
```

You now have two applications running on your EKS cluster - NGINX and Apache!

---

## When to Use Helm

Helm shines for:

- **Complex applications** with many interdependent resources
- **Community charts** from trusted sources (check licenses!)
- **Environment-specific configurations** using values files
- **Release management** with rollback capabilities

For simple deployments like web servers, manual manifests are often clearer and easier to maintain.

### Example: Using Helm with Community Charts

Here's how you might use Helm with a community-maintained chart:

```bash
# Add a repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Install with custom values
helm install my-prometheus prometheus-community/prometheus \
  --set server.persistentVolume.enabled=false

# List releases
helm list

# Upgrade a release
helm upgrade my-prometheus prometheus-community/prometheus --set server.replicas=2

# View history
helm history my-prometheus

# Rollback if needed
helm rollback my-prometheus 1

# Uninstall
helm uninstall my-prometheus
```

---

## Project Structure

For reference, here's the complete file structure for this article:

```
k8s-manifests/
├── README.md                    # Usage instructions
├── nginx-deployment.yaml        # NGINX Deployment manifest
├── nginx-service.yaml           # NGINX LoadBalancer Service manifest
├── apache-deployment.yaml       # Apache Deployment manifest
└── apache-service.yaml          # Apache LoadBalancer Service manifest
```

All files are available in the series repository.

---

## Clean Up

Let's clean up the resources we created:

```bash
# Navigate to the k8s-manifests directory
cd k8s-manifests

# Delete NGINX deployment and service
kubectl delete -f nginx-service.yaml
kubectl delete -f nginx-deployment.yaml

# Delete Apache deployment and service
kubectl delete -f apache-service.yaml
kubectl delete -f apache-deployment.yaml
```

Verify everything is removed:

```bash
kubectl get all
```

---

## Summary

In this article, we covered:

- **Kubernetes Deployments** — Declare and manage your application Pods
- **Pods** — The smallest deployable units in Kubernetes
- **Services** — Provide stable endpoints and load balancing
- **LoadBalancer Services** — Automatically provision AWS ELBs for external access
- **Scaling** — Easily scale applications up or down
- **Helm** — Package manager for Kubernetes that simplifies deployments

---

## Key Takeaways

- Deployments manage Pods and ensure your desired state is maintained
- Services provide stable networking for ephemeral Pods
- LoadBalancer Services integrate with AWS to provision ELBs automatically
- Helm simplifies deploying complex applications with a single command
- Always use resource requests and limits for production workloads
- Labels are crucial for connecting Deployments and Services

---

## What's Next?

In **Part 5**, we'll introduce **GitOps** and manage our deployments using **Argo CD**. You'll learn:
- What GitOps is and why it matters
- Setting up Argo CD on EKS
- Deploying applications from Git repositories
- Automated sync and self-healing deployments

Stay tuned!

---

## Resources

- [Kubernetes Deployments Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Helm Documentation](https://helm.sh/docs/)
- [Amazon EKS Load Balancing](https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)

---

*Found this article helpful? Follow along with the series and drop your questions in the comments!*
