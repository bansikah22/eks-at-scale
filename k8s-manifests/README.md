# Kubernetes Manifests for Amazon EKS Series

This directory contains Kubernetes manifest files used in the Amazon EKS at Scale series.

## Files

| File | Description | Article |
|------|-------------|---------|
| `nginx-deployment.yaml` | NGINX Deployment with 3 replicas | Part 4 |
| `nginx-service.yaml` | LoadBalancer Service for NGINX | Part 4 |
| `apache-deployment.yaml` | Apache Deployment with 3 replicas | Part 4 |
| `apache-service.yaml` | LoadBalancer Service for Apache | Part 4 |

## Usage

### Prerequisites

Ensure your EKS cluster is running and kubectl is configured:

```bash
# Verify cluster connection
kubectl get nodes
```

### Deploy NGINX Application

```bash
# Apply the deployment and service
kubectl apply -f nginx-deployment.yaml -f nginx-service.yaml

# Check pods
kubectl get pods -l app=nginx

# Check service (wait for EXTERNAL-IP)
kubectl get svc nginx-service --watch
```

### Deploy Apache Application

```bash
# Apply the deployment and service
kubectl apply -f apache-deployment.yaml -f apache-service.yaml

# Check pods
kubectl get pods -l app=apache

# Check service (wait for EXTERNAL-IP)
kubectl get svc apache-service --watch
```

### Access the Applications

> **Note:** AWS ELB DNS propagation can take 2-5 minutes. If you get a DNS resolution error, wait and retry.

```bash
# Test NGINX
LB_URL=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$LB_URL

# Test Apache
LB_URL=$(kubectl get svc apache-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$LB_URL
```

### Clean Up

```bash
# Delete all resources
kubectl delete -f nginx-service.yaml -f nginx-deployment.yaml
kubectl delete -f apache-service.yaml -f apache-deployment.yaml
```

## Notes

- Both deployments use official Docker Hub images (`nginx:1.25` and `httpd:2.4`)
- Each deployment includes resource requests and limits (best practice)
- Services are of type `LoadBalancer` which provisions AWS ELBs automatically
