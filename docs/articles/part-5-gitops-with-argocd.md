# Amazon EKS Series - Part 5: GitOps with Argo CD on EKS

---
title: Amazon EKS Series - Part 5: GitOps with Argo CD on EKS
published: true
description: Learn how to implement GitOps on Amazon EKS using Argo CD for automated, declarative application deployments.
tags: aws, kubernetes, eks, gitops, argocd
series: Amazon EKS at Scale
cover_image: 
---

## Introduction

Welcome to the final part of the **Amazon EKS at Scale** series!

In [Part 4](./part-4-deploying-applications-on-eks.md), we deployed applications manually using `kubectl` and explored Helm for package management. While these approaches work, they have limitations:

- **Manual processes** are error-prone
- **No audit trail** of who deployed what and when
- **Drift** can occur between what's in Git and what's running
- **Rollbacks** require remembering previous configurations

In this article, we'll solve these problems by implementing **GitOps** with **Argo CD**. You'll learn how to:

- Understand GitOps principles and benefits
- Install and configure Argo CD on EKS
- Deploy a real application (CloudNotes) using GitOps
- Enable automated sync and self-healing
- Observe deployments through the Argo CD UI

**Prerequisites:**
- EKS cluster from Part 3 running
- `kubectl` configured and working
- Basic understanding of Kubernetes Deployments and Services

```bash
# Verify cluster connection
kubectl get nodes
```

---

## What Is GitOps?

**GitOps** is an operational framework that uses Git as the single source of truth for declarative infrastructure and applications.

### The Core Idea

Instead of running commands to deploy applications:

```bash
# Traditional approach
kubectl apply -f deployment.yaml
```

With GitOps, you:
1. **Commit** your desired state to Git
2. **An agent** (like Argo CD) watches the repository
3. **Automatically** applies changes to the cluster

### GitOps Principles

| Principle | Description |
|-----------|-------------|
| **Declarative** | The entire system is described declaratively in Git |
| **Versioned** | All changes are tracked with Git history |
| **Automated** | Approved changes are applied automatically |
| **Self-healing** | The system corrects drift from the desired state |

### Desired State vs Actual State

GitOps continuously compares:

- **Desired State**: What's defined in your Git repository
- **Actual State**: What's currently running in the cluster

If they differ, GitOps reconciles by applying the desired state.

```
┌─────────────────┐         ┌─────────────────┐
│   Git Repo      │         │   Kubernetes    │
│  (Desired State)│◄───────►│  (Actual State) │
└─────────────────┘  Sync   └─────────────────┘
         │                           │
         └───────── Argo CD ─────────┘
              (Reconciliation)
```

---

## Why GitOps Matters for EKS

GitOps is particularly powerful for Amazon EKS environments:

### 1. Auditability

Every deployment is a Git commit:
- Who made the change?
- When was it made?
- What exactly changed?
- Why? (commit message)

This is invaluable for compliance and debugging.

### 2. Easy Rollbacks

Made a bad deployment? Simply revert the Git commit:

```bash
git revert HEAD
git push
```

Argo CD automatically rolls back the cluster.

### 3. Team Collaboration

- Use pull requests for deployment reviews
- Enforce approval workflows
- No need to share cluster credentials widely

### 4. Reduced Human Error

- No more typos in `kubectl` commands
- No forgotten flags or wrong contexts
- Consistent deployments every time

### 5. Disaster Recovery

Your entire cluster state is in Git. If you lose the cluster:
1. Create a new EKS cluster
2. Install Argo CD
3. Point to your Git repos
4. Everything redeploys automatically

---

## Introducing Argo CD

**Argo CD** is a declarative, GitOps continuous delivery tool for Kubernetes.

### How Argo CD Works

1. **Runs inside your cluster** as a set of Kubernetes controllers
2. **Watches Git repositories** for changes
3. **Compares** desired state (Git) with actual state (cluster)
4. **Syncs** the cluster to match Git (manually or automatically)
5. **Reports** status through UI, CLI, and API

### Argo CD vs kubectl/Helm

| Aspect | kubectl/Helm | Argo CD |
|--------|--------------|---------|
| Deployment trigger | Manual command | Git commit |
| State tracking | None | Continuous |
| Drift detection | Manual | Automatic |
| Rollback | Manual | Git revert |
| Audit trail | External logging | Git history |
| UI | None (CLI only) | Web dashboard |

### Argo CD Architecture (Simplified)

```
┌──────────────────────────────────────────────────────┐
│                    Argo CD                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ API Server  │  │ Repo Server │  │ Application │  │
│  │  (UI/CLI)   │  │ (Git sync)  │  │ Controller  │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
└──────────────────────────────────────────────────────┘
         │                  │                │
         ▼                  ▼                ▼
    ┌─────────┐       ┌──────────┐    ┌───────────────┐
    │ Web UI  │       │   Git    │    │  Kubernetes   │
    │   CLI   │       │  Repos   │    │    Cluster    │
    └─────────┘       └──────────┘    └───────────────┘
```

- **API Server**: Provides the web UI and CLI interface
- **Repo Server**: Clones and caches Git repositories
- **Application Controller**: Monitors applications and syncs state

---

## Installing Argo CD on EKS

Let's install Argo CD on our EKS cluster.

### Step 1: Create Namespace

```bash
kubectl create namespace argocd
```

### Step 2: Install Argo CD

We'll use the official installation manifests:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs:
- Argo CD API Server
- Argo CD Repo Server
- Argo CD Application Controller
- Redis (for caching)
- Dex (for authentication)

### Step 3: Wait for Pods to be Ready

```bash
kubectl get pods -n argocd --watch
```

Wait until all pods show `Running` status (1-2 minutes):

```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          60s
argocd-applicationset-controller-5f7b9d6b4-xxxxx   1/1     Running   0          60s
argocd-dex-server-6dcf4f8d4b-xxxxx                 1/1     Running   0          60s
argocd-notifications-controller-5c8d7b4c6d-xxxxx   1/1     Running   0          60s
argocd-redis-6976fc7dfc-xxxxx                      1/1     Running   0          60s
argocd-repo-server-7b8d7c4f5-xxxxx                 1/1     Running   0          60s
argocd-server-5f8d7c4f5b-xxxxx                     1/1     Running   0          60s
```

Press `Ctrl+C` to stop watching.

### Step 4: Expose the Argo CD Server

For this tutorial, we'll use port-forwarding. In production, you'd use an Ingress or LoadBalancer.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep this terminal open. The Argo CD UI is now available at: `https://localhost:8080`

### Step 5: Get the Initial Admin Password

Argo CD generates a random password for the `admin` user:

```bash
argocd admin initial-password -n argocd
```

Or using kubectl:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Save this password - you'll need it to log in.

### Step 6: Access the UI

1. Open `https://localhost:8080` in your browser
2. Accept the self-signed certificate warning
3. Log in with:
   - **Username**: `admin`
   - **Password**: (from Step 5)

You should see the Argo CD dashboard - currently empty since we haven't deployed any applications yet.

### Step 7: Install Argo CD CLI (Optional but Recommended)

```bash
# Linux
wget https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64

# macOS
brew install argocd

# Verify installation
argocd version
```

Login via CLI:

```bash
argocd login localhost:8080 --username admin --password <your-password> --insecure
```

---

## The CloudNotes Application

For this tutorial, we'll deploy **CloudNotes** - a full-stack ToDo Notes application.

### Application Overview

- **Repository**: [github.com/bansikah22/cloudnotes](https://github.com/bansikah22/cloudnotes)
- **Frontend**: React + Vite + TypeScript (served by Nginx)
- **Backend**: Node.js + Express + TypeScript
- **Images**: Available on Docker Hub

### Why CloudNotes?

- **Real application** - Not just a demo
- **Multiple services** - Frontend and Backend
- **Already containerized** - Docker images ready
- **Demonstrates GitOps** - Multiple manifests to sync

---

## Preparing Kubernetes Manifests for GitOps

For GitOps, we need Kubernetes manifests stored in a Git repository. We'll create them in our EKS series repository.

### Recommended Structure

```
argocd/
├── README.md
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

This separation keeps:
- **Argo CD configuration** in `applications/`
- **Application manifests** in `cloudnotes/`

---

## Writing the Kubernetes Manifests

Let's create the manifests for CloudNotes.

### Namespace

Create `argocd/cloudnotes/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudnotes
  labels:
    app: cloudnotes
    managed-by: argocd
```

### Backend Deployment

Create `argocd/cloudnotes/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudnotes-backend
  namespace: cloudnotes
  labels:
    app: cloudnotes
    component: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudnotes
      component: backend
  template:
    metadata:
      labels:
        app: cloudnotes
        component: backend
    spec:
      containers:
      - name: backend
        image: bansikah/cloudnotes-backend:latest
        ports:
        - containerPort: 5000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "5000"
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /api/health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /api/health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 10
```

### Backend Service

Create `argocd/cloudnotes/backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloudnotes-backend
  namespace: cloudnotes
  labels:
    app: cloudnotes
    component: backend
spec:
  type: ClusterIP
  selector:
    app: cloudnotes
    component: backend
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
```

> **Note:** The backend uses `ClusterIP` (internal only). The frontend's Nginx will proxy API requests to this service.

### Nginx ConfigMap (API Proxy)

The frontend is a static React app served by Nginx. When your browser makes API calls to `/api/*`, we need Nginx to proxy those requests to the backend service.

Create `argocd/cloudnotes/nginx-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: cloudnotes
  labels:
    app: cloudnotes
    component: frontend
data:
  default.conf: |
    server {
        listen 80;

        # Serve static frontend files
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
            try_files $uri $uri/ /index.html;
        }

        # Proxy API requests to backend service
        location /api/ {
            proxy_pass http://cloudnotes-backend:5000/api/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
```

**Why is this needed?**
- The frontend runs in your **browser** (external)
- The backend is **ClusterIP** (internal only)
- Without the proxy, API calls from the browser would fail
- Nginx acts as a reverse proxy, forwarding `/api/*` requests to the backend

### Frontend Deployment

Create `argocd/cloudnotes/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudnotes-frontend
  namespace: cloudnotes
  labels:
    app: cloudnotes
    component: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudnotes
      component: frontend
  template:
    metadata:
      labels:
        app: cloudnotes
        component: frontend
    spec:
      containers:
      - name: frontend
        image: bansikah/cloudnotes-frontend:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/http.d/default.conf
          subPath: default.conf
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```

> **Important:** The `volumeMounts` section mounts our custom Nginx config, enabling the API proxy to the backend.

### Frontend Service

Create `argocd/cloudnotes/frontend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloudnotes-frontend
  namespace: cloudnotes
  labels:
    app: cloudnotes
    component: frontend
spec:
  type: LoadBalancer
  selector:
    app: cloudnotes
    component: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

---

## Creating an Argo CD Application

Now we create the Argo CD Application resource that tells Argo CD what to deploy and where.

### Understanding the Application CRD

An Argo CD **Application** is a Custom Resource that defines:
- **Source**: Where to get the manifests (Git repo, path, branch)
- **Destination**: Where to deploy (cluster, namespace)
- **Sync Policy**: How to handle updates (manual/automatic)

### The Application Manifest

Create `argocd/applications/cloudnotes.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudnotes
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: https://github.com/bansikah22/eks-at-scale.git
    targetRevision: master
    path: argocd/cloudnotes
  
  destination:
    server: https://kubernetes.default.svc
    namespace: cloudnotes
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Breaking Down the Application

| Field | Purpose |
|-------|---------|
| `metadata.name` | Name of the application in Argo CD |
| `metadata.namespace` | Must be `argocd` |
| `spec.project` | Argo CD project (use `default` for now) |
| `source.repoURL` | Git repository URL |
| `source.targetRevision` | Branch, tag, or commit to track |
| `source.path` | Path to manifests within the repo |
| `destination.server` | Target cluster (in-cluster uses this URL) |
| `destination.namespace` | Target namespace for resources |
| `syncPolicy.automated` | Enable automatic sync |
| `syncPolicy.automated.prune` | Delete resources removed from Git |
| `syncPolicy.automated.selfHeal` | Revert manual changes to cluster |

---

## Deploying via GitOps

Now let's see GitOps in action!

### Step 1: Commit and Push the Manifests

First, ensure all your manifests are committed to Git:

```bash
cd /path/to/eks-at-scale
git add argocd/
git commit -m "Add CloudNotes GitOps manifests"
git push origin master
```

### Step 2: Apply the Argo CD Application

```bash
kubectl apply -f argocd/applications/cloudnotes.yaml
```

Output:

```
application.argoproj.io/cloudnotes created
```

### Step 3: Watch Argo CD Sync

Check the application status:

```bash
kubectl get applications -n argocd
```

Output:

```
NAME         SYNC STATUS   HEALTH STATUS
cloudnotes   Synced        Healthy
```

Or use the Argo CD CLI:

```bash
argocd app get cloudnotes
```

### Step 4: Observe in the UI

Open the Argo CD UI (`https://localhost:8080`) and you'll see:

1. The **cloudnotes** application card
2. **Sync Status**: Synced (green)
3. **Health Status**: Healthy (green heart)

Click on the application to see:
- All Kubernetes resources created
- Resource relationships (Deployment -> ReplicaSet -> Pods)
- Real-time sync status

### Step 5: Verify the Deployment

Check that CloudNotes is running:

```bash
# Check pods
kubectl get pods -n cloudnotes

# Check services
kubectl get svc -n cloudnotes
```

Expected output:

```
NAME                                   READY   STATUS    RESTARTS   AGE
cloudnotes-backend-xxxxx-xxxxx         1/1     Running   0          2m
cloudnotes-backend-xxxxx-xxxxx         1/1     Running   0          2m
cloudnotes-frontend-xxxxx-xxxxx        1/1     Running   0          2m
cloudnotes-frontend-xxxxx-xxxxx        1/1     Running   0          2m
```

### Step 6: Access the Application

Get the LoadBalancer URL:

```bash
kubectl get svc cloudnotes-frontend -n cloudnotes --watch
```

Once you have the EXTERNAL-IP:

```bash
LB_URL=$(kubectl get svc cloudnotes-frontend -n cloudnotes -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "CloudNotes URL: http://$LB_URL"
```

Open the URL in your browser to see the CloudNotes application!

---

## Automated Sync and Self-Healing

This is where GitOps truly shines.

### Automated Sync in Action

Let's make a change to our manifests and watch Argo CD automatically sync.

**1. Update the replica count:**

Edit `argocd/cloudnotes/frontend-deployment.yaml`:

```yaml
spec:
  replicas: 3  # Changed from 2 to 3
```

**2. Commit and push:**

```bash
git add argocd/cloudnotes/frontend-deployment.yaml
git commit -m "Scale frontend to 3 replicas"
git push origin master
```

**3. Watch Argo CD sync:**

Within 3 minutes (default sync interval), Argo CD will:
1. Detect the Git change
2. Compare with cluster state
3. Apply the new configuration

```bash
kubectl get pods -n cloudnotes -l component=frontend --watch
```

You'll see a third frontend pod being created automatically!

### Self-Healing in Action

Self-healing reverts unauthorized changes made directly to the cluster.

**1. Manually delete a pod:**

```bash
kubectl delete pod -n cloudnotes -l component=frontend --wait=false
```

**2. Watch Argo CD restore it:**

```bash
kubectl get pods -n cloudnotes -l component=frontend --watch
```

Within seconds, Argo CD detects the drift and recreates the pod to match the desired state (3 replicas).

**3. Try scaling manually:**

```bash
kubectl scale deployment cloudnotes-frontend -n cloudnotes --replicas=1
```

Watch as Argo CD reverts this change back to 3 replicas (as defined in Git).

### Why This Matters

- **Accidental deletions** are automatically recovered
- **Unauthorized changes** are reverted
- **Cluster state** always matches Git
- **No manual intervention** required

---

## Observing the Deployment

### Argo CD UI Features

The Argo CD web interface provides:

**Application View:**
- Visual resource tree showing all Kubernetes objects
- Health status for each resource
- Sync status (in sync, out of sync, unknown)
- Recent sync history

**Resource Details:**
- Click any resource to see its YAML
- View events and logs
- See relationships between resources

**Sync Operations:**
- Manual sync button
- Sync with prune option
- Rollback to previous versions

### Useful CLI Commands

```bash
# List all applications
argocd app list

# Get detailed application info
argocd app get cloudnotes

# View application history
argocd app history cloudnotes

# Manually trigger sync
argocd app sync cloudnotes

# View application logs
argocd app logs cloudnotes

# Rollback to previous version
argocd app rollback cloudnotes 1
```

### Checking Health Status

```bash
# Quick status check
kubectl get applications -n argocd

# Detailed health
argocd app get cloudnotes --show-operation
```

Health statuses:
- **Healthy**: All resources are healthy
- **Progressing**: Resources are being updated
- **Degraded**: Some resources have issues
- **Suspended**: Application is paused
- **Missing**: Resources don't exist yet

---

## GitOps Workflow Summary

Here's the complete GitOps workflow we've implemented:

```
Developer                    Git                      Argo CD                  EKS Cluster
    │                         │                          │                          │
    │  1. Edit manifests      │                          │                          │
    │─────────────────────────►                          │                          │
    │                         │                          │                          │
    │  2. git commit & push   │                          │                          │
    │─────────────────────────►                          │                          │
    │                         │                          │                          │
    │                         │  3. Detect changes       │                          │
    │                         │◄─────────────────────────│                          │
    │                         │                          │                          │
    │                         │                          │  4. Apply manifests      │
    │                         │                          │─────────────────────────►│
    │                         │                          │                          │
    │                         │                          │  5. Report status        │
    │                         │                          │◄─────────────────────────│
    │                         │                          │                          │
    │  6. View in UI/CLI      │                          │                          │
    │◄─────────────────────────────────────────────────────                         │
```

---

## Clean Up

When you're done experimenting:

### Delete the Argo CD Application

```bash
# This will delete all CloudNotes resources
argocd app delete cloudnotes --cascade

# Or using kubectl
kubectl delete application cloudnotes -n argocd
```

### Uninstall Argo CD (Optional)

```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```

### Delete the EKS Cluster

If you're done with the entire series:

```bash
cd eks-terraform
terraform destroy --auto-approve
```

---

## Summary

In this article, we covered:

- **GitOps Fundamentals** - Git as the single source of truth
- **Why GitOps Matters** - Auditability, rollbacks, collaboration, disaster recovery
- **Argo CD** - A powerful GitOps tool for Kubernetes
- **Installation** - Setting up Argo CD on EKS
- **Application Deployment** - Deploying CloudNotes via GitOps
- **Automated Sync** - Changes in Git automatically applied
- **Self-Healing** - Drift detection and automatic correction

---

## Key Takeaways

- **Git is the source of truth** - All desired state lives in version control
- **Argo CD automates deployments** - No more manual `kubectl apply`
- **Changes are auditable** - Every deployment is a Git commit
- **Self-healing ensures consistency** - The cluster always matches Git
- **Rollbacks are simple** - Just revert the Git commit
- **Team collaboration improves** - Use PRs for deployment reviews

---

## Series Conclusion

Congratulations! You've completed the **Amazon EKS at Scale** series!

Over five articles, you've learned:

| Part | Topic | Key Skills |
|------|-------|------------|
| 1 | Introduction to EKS | Understanding managed Kubernetes, worker nodes |
| 2 | EKS Architecture | Control plane, networking, IAM integration |
| 3 | Provisioning with Terraform | Infrastructure as Code, VPC, EKS modules |
| 4 | Deploying Applications | Deployments, Services, LoadBalancers |
| 5 | GitOps with Argo CD | Declarative deployments, automated sync, self-healing |

You now have a complete, production-ready workflow:

```
Infrastructure (Terraform) → Cluster (EKS) → Applications (GitOps/Argo CD)
```

### What's Next?

Here are some areas to explore further:

- **Secrets Management** - HashiCorp Vault, AWS Secrets Manager, Sealed Secrets
- **CI/CD Integration** - GitHub Actions, Jenkins, GitLab CI with Argo CD
- **Multi-Cluster** - Argo CD managing multiple EKS clusters
- **Advanced Argo CD** - ApplicationSets, App of Apps pattern
- **Monitoring** - Prometheus, Grafana, CloudWatch Container Insights
- **Service Mesh** - Istio, AWS App Mesh for advanced networking

---

## Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://opengitops.dev/)
- [CloudNotes Repository](https://github.com/bansikah22/cloudnotes)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

## Thank You!

Thank you for following along with this series! 

If you found it helpful:
- Star the [GitHub repository](https://github.com/bansikah22/eks-at-scale)
- Share with others learning Kubernetes
- Drop your questions and feedback in the comments

Happy deploying!

---

*This concludes the Amazon EKS at Scale series. Stay tuned for more DevOps and cloud-native content!*
