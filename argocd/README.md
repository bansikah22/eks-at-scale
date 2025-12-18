# ArgoCD GitOps Configuration

This directory contains ArgoCD configuration and Kubernetes manifests for GitOps-based deployments.

## Structure

```
argocd/
├── README.md                    # This file
├── applications/
│   └── cloudnotes.yaml          # Argo CD Application resource
└── cloudnotes/
    ├── namespace.yaml           # Kubernetes namespace
    ├── nginx-configmap.yaml     # Nginx config for API proxy
    ├── backend-deployment.yaml  # Backend deployment
    ├── backend-service.yaml     # Backend service (ClusterIP)
    ├── frontend-deployment.yaml # Frontend deployment
    └── frontend-service.yaml    # Frontend service (LoadBalancer)
```

## Prerequisites

1. EKS cluster running (see Part 3)
2. kubectl configured
3. Argo CD installed on the cluster

## Installing Argo CD

```bash
# Create namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl get pods -n argocd --watch
```

## Accessing Argo CD UI

```bash
# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Open `https://localhost:8080` and login with username `admin`.

## Deploying CloudNotes

### Option 1: Using kubectl

```bash
kubectl apply -f applications/cloudnotes.yaml
```

### Option 2: Using Argo CD CLI

```bash
argocd app create cloudnotes \
  --repo https://github.com/bansikah22/eks-at-scale.git \
  --path argocd/cloudnotes \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace cloudnotes \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Verifying Deployment

```bash
# Check Argo CD application status
kubectl get applications -n argocd

# Check CloudNotes pods
kubectl get pods -n cloudnotes

# Get LoadBalancer URL
kubectl get svc cloudnotes-frontend -n cloudnotes
```

## Testing Self-Healing

```bash
# Delete a pod manually
kubectl delete pod -n cloudnotes -l component=frontend

# Watch Argo CD restore it
kubectl get pods -n cloudnotes --watch
```

## Clean Up

```bash
# Delete the application (and all resources)
kubectl delete application cloudnotes -n argocd

# Or using Argo CD CLI
argocd app delete cloudnotes --cascade
```

## Related Articles

- [Part 5: GitOps with Argo CD](../docs/articles/part-5-gitops-with-argocd.md)
