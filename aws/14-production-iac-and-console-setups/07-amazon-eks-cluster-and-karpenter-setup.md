# Production Amazon EKS Cluster & Karpenter Autoscaling: Terraform & Console Setup

Amazon Elastic Kubernetes Service (EKS) provides a managed, high-availability Kubernetes control plane. In modern production environments, legacy Cluster Autoscaler is replaced by **Karpenter**, an intelligent, just-in-time node provisioning engine. This guide implements an enterprise EKS cluster with private networking, AWS VPC CNI, and Karpenter autoscaling prerequisites.

---

## Architecture Flow

```
[ Amazon EKS Control Plane ] (Managed by AWS across 3 AZs: K8s 1.30)
            |
            +---------------------------------------+
            | (Managed Worker Nodes)                | (Karpenter Just-in-Time Nodes)
            v                                       v
[ System Node Group: 2x t4g.medium ]      [ Dynamic Karpenter NodePool ]
- CoreDNS, kube-proxy, VPC CNI            - Provisioned in 45s based on Pod requests
- Karpenter Controller pod                - Spot consolidation & Bin-packing
```

---

## Part 1: Production Terraform Implementation

### 1.1 `variables.tf`
```hcl
variable "aws_region" {
  description = "Target AWS Region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "production"
}

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
  default     = "production-core-eks"
}

variable "kubernetes_version" {
  description = "Kubernetes control plane version"
  type        = string
  default     = "1.30"
}

variable "vpc_id" {
  description = "VPC ID where EKS nodes will be deployed"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private Subnet IDs across multiple AZs for worker nodes"
  type        = list(string)
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. IAM Role for EKS Cluster Control Plane
# ------------------------------------------------------------------------------
resource "aws_iam_role" "cluster_role" {
  name_prefix = "${var.cluster_name}-controlplane-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "eks.amazonaws.com" }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "cluster_policy" {
  role       = aws_iam_role.cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

# ------------------------------------------------------------------------------
# 2. Amazon EKS Cluster Control Plane
# ------------------------------------------------------------------------------
resource "aws_eks_cluster" "eks" {
  name     = var.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.cluster_role.arn

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true # Can restrict to CIDRs via public_access_cidrs
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy
  ]

  tags = {
    Name        = var.cluster_name
    Environment = var.environment
    "karpenter.sh/discovery" = var.cluster_name
  }
}

# ------------------------------------------------------------------------------
# 3. OIDC Identity Provider (Mandatory for IRSA & Karpenter)
# ------------------------------------------------------------------------------
data "tls_certificate" "eks" {
  url = aws_eks_cluster.eks.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "oidc" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.eks.identity[0].oidc[0].issuer
}

# ------------------------------------------------------------------------------
# 4. IAM Role for Managed Worker Nodes
# ------------------------------------------------------------------------------
resource "aws_iam_role" "node_role" {
  name_prefix = "${var.cluster_name}-node-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "ec2.amazonaws.com" }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "worker_node_policy" {
  role       = aws_iam_role.node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "cni_policy" {
  role       = aws_iam_role.node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "ecr_readonly" {
  role       = aws_iam_role.node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_role_policy_attachment" "ssm_core" {
  role       = aws_iam_role.node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# ------------------------------------------------------------------------------
# 5. Core Managed Node Group (Hosts Karpenter, CoreDNS, Ingress Controllers)
# ------------------------------------------------------------------------------
resource "aws_eks_node_group" "system_nodes" {
  cluster_name    = aws_eks_cluster.eks.name
  node_group_name = "${var.cluster_name}-system-nodes"
  node_role_arn   = aws_iam_role.node_role.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 5
  }

  instance_types = ["t4g.medium"] # ARM64 Graviton
  capacity_type  = "ON_DEMAND"

  update_config {
    max_unavailable = 1
  }

  tags = {
    Name = "${var.cluster_name}-system-node"
    "karpenter.sh/discovery" = var.cluster_name
  }

  depends_on = [
    aws_iam_role_policy_attachment.worker_node_policy,
    aws_iam_role_policy_attachment.cni_policy,
    aws_iam_role_policy_attachment.ecr_readonly
  ]
}

# ------------------------------------------------------------------------------
# 6. Karpenter Controller IAM Role & Interruption Queue
# ------------------------------------------------------------------------------
resource "aws_sqs_queue" "karpenter_interruption" {
  name                      = "${var.cluster_name}-karpenter-interruption"
  message_retention_seconds = 300
}

resource "aws_iam_role" "karpenter_controller" {
  name_prefix = "karpenter-controller-${var.cluster_name}-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.oidc.arn
        }
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.oidc.url, "https://", "")}:sub" = "system:serviceaccount:karpenter:karpenter"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "karpenter_controller_policy" {
  name_prefix = "karpenter-policy-"
  role        = aws_iam_role.karpenter_controller.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:CreateLaunchTemplate",
          "ec2:CreateFleet",
          "ec2:RunInstances",
          "ec2:CreateTags",
          "ec2:TerminateInstances",
          "ec2:DescribeInstances",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeInstanceTypes",
          "ec2:DescribeInstanceTypeOfferings",
          "ec2:DescribeAvailabilityZones",
          "ssm:GetParameter"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "sqs:DeleteMessage",
          "sqs:GetQueueUrl",
          "sqs:ReceiveMessage"
        ]
        Resource = aws_sqs_queue.karpenter_interruption.arn
      },
      {
        Effect   = "Allow"
        Action   = "iam:PassRole"
        Resource = aws_iam_role.node_role.arn
      }
    ]
  })
}
```

### 1.3 `outputs.tf`
```hcl
output "cluster_name" {
  description = "The name of the EKS cluster"
  value       = aws_eks_cluster.eks.name
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane API"
  value       = aws_eks_cluster.eks.endpoint
}

output "cluster_certificate_authority_data" {
  description = "Base64 encoded certificate data required to communicate with the cluster"
  value       = aws_eks_cluster.eks.certificate_authority[0].data
}

output "karpenter_role_arn" {
  description = "IAM Role ARN for the Karpenter controller"
  value       = aws_iam_role.karpenter_controller.arn
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create IAM Role for EKS Cluster Control Plane
1. Open **IAM Console** -> **Roles** -> **Create role**.
2. Select **AWS service** -> Use case: **EKS - Cluster**.
3. AWS will automatically attach the `AmazonEKSClusterPolicy`.
4. Role name: `production-eks-cluster-role`. Click **Create role**.

---

### Step 2: Create IAM Role for EKS Worker Nodes
1. In IAM Console, click **Create role**.
2. Select **AWS service** -> Use case: **EC2**.
3. Attach the following 4 policies:
   * `AmazonEKSWorkerNodePolicy`
   * `AmazonEKS_CNI_Policy`
   * `AmazonEC2ContainerRegistryReadOnly`
   * `AmazonSSMManagedInstanceCore`
4. Role name: `production-eks-node-role`. Click **Create role**.

---

### Step 3: Create the EKS Cluster
1. Open the **Amazon EKS Console** (`https://console.aws.amazon.com/eks/`).
2. Click **Add cluster** -> **Create**.
3. **Configure cluster:**
   * Name: `production-core-eks`.
   * Kubernetes version: `1.30`.
   * Cluster service role: Select `production-eks-cluster-role`. Click **Next**.
4. **Specify networking:**
   * VPC: Select `production-core-vpc`.
   * Subnets: Select all **Private Application Subnets**.
   * Cluster endpoint access: Select **Public and private**. Click **Next**.
5. **Configure logging:** Enable `API server`, `Audit`, `Authenticator`. Click **Next**, then **Create**.
   *(Control plane takes ~8 to 12 minutes to provision)*.

---

### Step 4: Associate OIDC Provider for IAM Roles for Service Accounts (IRSA)
1. Once active, select your cluster -> **Overview** tab.
2. Note the **OpenID Connect provider URL**.
3. Run from terminal:
   ```bash
   eksctl utils associate-iam-oidc-provider --cluster production-core-eks --approve
   ```

---

### Step 5: Add Managed Node Group
1. In EKS Console, select your cluster -> **Compute** tab.
2. Under **Node groups**, click **Add node group**.
3. Name: `production-system-nodes`.
4. Node IAM role: Select `production-eks-node-role`. Click **Next**.
5. **Set compute and scaling configuration:**
   * AMI type: **Amazon Linux 2023 (ARM64)** or (x86_64).
   * Capacity type: **On-Demand**.
   * Instance types: `t4g.medium`.
   * Scaling: Min: `2`, Desired: `2`, Max: `5`. Click **Next**.
6. Subnets: Ensure only **Private Application Subnets** are selected. Click **Next**, then **Create**.

---

### Step 6: Cluster Verification via `kubectl`
```bash
# Update local kubeconfig to point to the new EKS cluster
aws eks update-kubeconfig --region us-east-1 --name production-core-eks

# Verify cluster nodes are in 'Ready' state
kubectl get nodes -o wide
# Output shows 2 nodes in Ready status across distinct AZs!
```
