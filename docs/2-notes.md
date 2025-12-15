# Amazon EKS — Common Use Cases (Official AWS Docs)

Amazon EKS provides a fully managed Kubernetes service that simplifies running containerized applications at scale. Below are common use cases where EKS is typically used to take advantage of its strengths.

## 1. Deploying High-Availability Applications

Ensure your applications stay available and resilient by using Elastic Load Balancing (ELB) across multiple AWS Availability Zones. This helps distribute traffic and tolerate failures without downtime.

## 2. Building Microservices Architectures

Use Kubernetes service discovery and AWS services like AWS Cloud Map or Amazon VPC Lattice to construct resilient microservices that can easily communicate with one another.

## 3. Automating Software Release Processes

Amazon EKS integrates well with CI/CD workflows to automate building, testing, and deploying applications. Declarative continuous deployment tools like Argo CD are often used for this purpose.

## 4. Running Serverless Applications

By combining EKS with AWS Fargate, you can run containers without managing servers — the infrastructure layer is abstracted away so you can focus on code.

## 5. Executing Machine Learning Workloads

EKS supports major machine learning frameworks like TensorFlow, MXNet, and PyTorch, and with GPU-enabled nodes, it can handle intensive training and inference workloads.

## 6. Deploying Consistently On-Premises and in the Cloud

Use Amazon EKS Anywhere or EKS Hybrid Nodes to run Kubernetes clusters both on-premises and in AWS, enabling hybrid cloud workloads with consistent tooling and configuration.

## 7. Running Cost-Effective Batch & Big Data Workloads

Leverage EC2 Spot Instances to run batch jobs or big data frameworks (e.g., Apache Hadoop, Spark) at a much lower cost by utilizing spare AWS capacity.

## 8. Managing AWS Resources from Kubernetes

Use AWS Controllers for Kubernetes (ACK) to create and manage AWS services (like S3 buckets or RDS databases) directly from Kubernetes APIs — bringing infrastructure and workloads under Kubernetes control.

## 9. Building Platform Engineering Abstractions

Platform teams can use tools like kro (Kube Resource Orchestrator) to define higher-level abstractions that bundle together resources and policies, simplifying developer workflows and enforcing standards.

## 10. Securing Applications & Ensuring Compliance

EKS works with AWS security tools such as IAM, VPC, and AWS KMS to help secure applications and maintain compliance with industry and regulatory standards.