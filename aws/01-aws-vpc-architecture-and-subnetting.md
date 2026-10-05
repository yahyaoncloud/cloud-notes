# AWS VPC Architecture, Subnetting & Network Segmentation

Amazon Virtual Private Cloud (VPC) provisions a logically isolated virtual network dedicated to your AWS account.

---

## 1. Multi-Tier Production VPC Design

```text
AWS Region (e.g. us-east-1)
└── VPC: 10.100.0.0/16
    ├── Availability Zone A (us-east-1a)
    │   ├── Public Subnet (10.100.1.0/24)   -> ALB, NAT Gateway A, Bastion
    │   ├── App Private Subnet (10.100.10.0/24) -> EC2, ECS Tasks (Fargate)
    │   └── Data Private Subnet (10.100.20.0/24) -> RDS Multi-AZ Primary
    │
    └── Availability Zone B (us-east-1b)
        ├── Public Subnet (10.100.2.0/24)   -> ALB, NAT Gateway B
        ├── App Private Subnet (10.100.11.0/24) -> EC2, ECS Tasks (Fargate)
        └── Data Private Subnet (10.100.21.0/24) -> RDS Multi-AZ Standby
```

---

## 2. Route Tables, Gateways & Security Boundaries

| Component | Scope | Functionality | Routing Rule |
| :--- | :--- | :--- | :--- |
| **Internet Gateway (IGW)** | VPC Level | Bi-directional Internet access for public subnets (1:1 Public IPv4 mapping). | `0.0.0.0/0 -> igw-xxxx` |
| **NAT Gateway (NAT GW)** | Subnet Level | Egress-only Internet access for private subnets (software updates, API calls). | `0.0.0.0/0 -> nat-xxxx` |
| **Security Groups (SG)** | ENI Level | **Stateful** firewall operating at the instance/ENI layer. Return traffic is automatically allowed. | Allow-only rules |
| **Network ACLs (NACL)** | Subnet Level | **Stateless** subnet boundary firewall. Explicit inbound AND outbound rules required. | Inbound & Outbound rules (Rule # evaluated in order) |

---

## 3. Terraform Multi-AZ VPC Implementation Snippet

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.100.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "production-vpc" }
}

resource "aws_subnet" "public_1a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.100.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet-1a" }
}

resource "aws_subnet" "private_app_1a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.100.10.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "private-app-1a" }
}
```
