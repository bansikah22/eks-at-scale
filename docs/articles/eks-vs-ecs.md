EKS vs ECS
Container Orchestration

 - Who monitors the servers available and resources?
 - What happens when a container crashes? 
 - How to scale up/down during peak traffic hours
 
 A container orchestration manages the lifecycle of a contain, it handles:
 - Deployment
 - Scaling
 - Restarting/Destroying
 - Loadbalancing

 Orchestrators 
 - K8s
 - Docker swarm 
 - AWS ECS and many others

 ECS
 what is ecs?
 - AWS proprietory container orchestrator
 - Acts as the control plane 
 - ECS is not responsible for physically running the containers

 EC2 Servers
 - Needs to install:
 - Container runtime
 - ECS Agent - Allows Ec2 control plane to communicate with ec2 instances
  EC2 Launch type
  - Underlying infrastructure has to be managed by you
  Fargate Launch type
  - Fargate follows a serverless architecture
  - Fargate will create servers on demand
  - Users don't need to provision/maintain ec2 servers
  - Only pay for what you use

  EKS
  To provision a cluster manually you will have to :
  - used kubeadmn to:
  - Intall control-plane processes on the ec2 
   - Api server
   - scheduler
   - controller manager
   - etcd

- Install worker process
  - kubelet
  - kube-proxy
  - Container runtime

Alternative is using EKS where AWS is incharge of 
- managing the control plane
- provisioning/mantaining master nodes
- Install control plane proecesses:
    - Api server
    - scheduler
    - controller manager
    - etcd
    
- scaling and backups

why EKS?
- Managing and scaling k8s is difficult
- properply securing k8s increases operational overhead
- tigh integration with other aws services
  - IAM
  - s3
  - secrets manager
  - Load balancers and much more

  Worker Nodes
- EKS does not manage worker nodes, it us up to you to set your worker nodes
- AWS provides several different options for managing worker nodes which are :
1. Self managed nodes
2. Managed Node groups
3. Fargate

 Self managed nodes
 - ec2 instances must be provisoned and managed by users
 - all k8s worker processes must be installed
   - kubelet
   - kube-proxy
   - Container runtime
- YOu must managed all updates and security patches
- register nodes with control plane

 Managed Node groups
    - AWS will provision and manage ec2 instances for you
    - AWS will handle updates and security patches
    - You still need to install k8s worker processes
    - kubelet
    - kube-proxy
    - Container runtime
    - Nodes are registered with control plane automatically
    
Fargate
    - Serverless compute engine for containers
    - No need to provision/manage ec2 instances
    - AWS handles all infrastructure management tasks
    - Only pay for what you use based on vCPU and memory resources

ecs vs eks comparison
ECS:
- AWS specific vendor lockedin 
- ecs follows a simplier architecture, faster onboarding,simpler api
- pricing- no cost for control plane, only pay for infra running applications(ec2, ebs ...)
EKS
- K8s - Opensource, , same api across all cloud providers(azure, gcp and so on)
- steep learning curve- learn k8s and eks specific features 
- Larger community
  - More support online
  - more tooling like helm, kustomize, argocd ..
- more expensive, control plane and worker nodes are paid for 
    
   When to us
    - ECS is a good choice if you want a fully managed container orchestration service that is tightly integrated with AWS services and you prefer a simpler setup without the complexity of managing a Kubernetes cluster.
    - EKS is a better choice if you need the flexibility and extensibility of Kubernetes, want to leverage its large ecosystem, or have existing workloads that are already running on Kubernetes.