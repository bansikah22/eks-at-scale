# Setting Up kubectl and eksctl for Amazon EKS

This guide covers everything you need to manage EKS clusters from your local machine: installing the CLI tools kubectl (Kubernetes CLI) and eksctl (EKS CLI).

## 1. Install kubectl (Kubernetes CLI)

kubectl is used to interact with your Kubernetes cluster, deploy applications, and inspect resources.

### Linux

```sh
# Download latest stable release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make it executable
chmod +x kubectl

# Move to a directory in PATH
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

### macOS

```sh
# Using Homebrew
brew install kubectl

# Verify installation
kubectl version --client
```

### Windows

```sh
# Using Chocolatey
choco install kubernetes-cli

# Verify installation
kubectl version --client
```

## 2. Install eksctl (EKS CLI)

eksctl is used to create and manage EKS clusters with simple commands.

### Linux & macOS

```sh
# Download latest release
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# Move binary to PATH
sudo mv /tmp/eksctl /usr/local/bin/

# Verify installation
eksctl version
```

### Windows

```sh
# Using Chocolatey
choco install eksctl

# Verify installation
eksctl version
```

## 3. Configure Access to Your EKS Cluster

Once both tools are installed:

- Create or identify your EKS cluster (or create a new one using eksctl):

```sh
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodes 2
```

- Update kubeconfig to allow kubectl to access your cluster:

```sh
aws eks --region us-east-1 update-kubeconfig --name my-cluster
```

- Test connection:

```sh
kubectl get nodes
```

If you see the worker nodes listed, everything is correctly set up.

## 4. Quick Tips

- Keep kubectl version compatible with your cluster. Minor version mismatches are okay.
- Use eksctl for easy cluster creation and deletion; manual cluster setup is more complex.
- Once configured, kubectl and eksctl allow full lifecycle management: create, scale, and delete clusters and workloads.