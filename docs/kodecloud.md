## What is EKS?

- A Managed kubernetes service .
- AWS manages the control plane
- Provisioning/maintaining master nodes
- Install Control plane processes
  - `API server`
  - `Scheduler`
  - `etcd`
- Scaling and backups of cluster
- Patching and updating control plane components

## Why EKS?

- Running and scaling kubernetes can be difficult
- Properly `securing` kubernetes increases operational overhead
- Tight integration with other AWS services 

### Worker Nodes

#### What about the worker nodes?

EKS does not manage worker nodes, it is up to you to setup the work nodes, and there are couple of ways we can do this which is : 
1. `Self-managed nodes`
2. `Managed node group`
3. `Fargate`

#### Idea about Self-managed Nodes
- Users must provision manually EC2 instances they want to use as worker nodes.
- All kubernetes worker processes must be installed
   - `Kubelet` 
   - `Kube-proxy`
   - `Container runtime`
- Updates and security patches are the user's responsibility
- Register node with Control-plane

#### Managed Node Groups
- Automates the provisioning and lifecycle management of EC2 nodes
- Managed nodes run EKS optimized images
- Streamline way to manage lifecycle of nodes using single AWS/EKS API call
  - `Create`
  - `Update`
  - `Terminate`

- Every node is part of an Auto Scaling group that's managed for you by EKS

#### Fargate
- Follows a `serverless` architecture
- Fargate will create worker nodes on demand
- No need to provision/maintain EC2 servers
- Based on container requirements Fargate will automatically select optimal EC2 sizing
- You only have to pay for what you use


## Creating EKS Cluster
#### Things we need 
- Cluster name, k8s version
- IAM role for cluster
  - provisioning nodes
  - Storage
  - secrets
- Select VPC & Subnets
- Define security group for cluster

### Creating Worker Nodes
- Create a Node group(A node group is a group nodes that will be specified to be used in our k8s environment).
- Select instance type
- Define min/max number of nodes
- Specify which EKS cluster you want to connect to

### Connecting to Cluster
To connect to EKS cluster: 
- Needs `kubectl` installed and we need to set
- `$ kubectl config set-cluster`

### Different Ways to Create Cluster
#### Methods to create EKS Cluster
1. `AWS Console`(longer process)
2. `EKSctl` using this cli tool to create the cluster, needs to be installed locally on your machine 
- `$eksctl create cluster`
provisions the security group, vpc and so ....
3. `IaC - Terraform/pulumi`

### Installing EKSCTL tool
Use [Install Eksctl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

### Demo
Make sure authentication is set us
- Commands to use 
```bash
eksctl --help

eksctl create --help

eksctl create cluster --help

eksctl create cluster -n cluster1 --nodegroup-name ng1 --region us-east-1 --node-type t2.micro --nodes 2

```
- Check created resources
- vpc
- subnets 
- EKS services 
- Compute -> nodegroup
- Instances

### Verify connection
```
kubectl config view

kubectl get nodes ## to verify nodes
```

### Clean up
```bash
eksctl delete cluster -n cluster1
```