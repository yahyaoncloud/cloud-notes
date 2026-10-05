# Production 3-Tier VPC & Networking: Complete Terraform IaC & Console Setup

This guide delivers the enterprise standard for network segmentation in AWS: a **Multi-AZ 3-Tier Virtual Private Cloud (VPC)** with Public Subnets, Private Application Subnets, Isolated Database Subnets, redundant NAT Gateways, and VPC Flow Logs.

It contains:
1. **Production Terraform Implementation:** Modular HCL with `variables.tf`, `main.tf`, and `outputs.tf`.
2. **AWS Management Console Walkthrough:** Exact step-by-step navigation and configuration instructions.

---

## Architecture Blueprint

```
VPC: 10.100.0.0/16 (us-east-1)
  ├── Availability Zone A (us-east-1a)
  │   ├── Public Subnet A:       10.100.1.0/24   -> ALB, NAT Gateway A
  │   ├── Private App Subnet A:  10.100.10.0/24  -> EC2 / ECS / EKS
  │   └── Private DB Subnet A:   10.100.20.0/24  -> RDS Aurora Primary
  │
  └── Availability Zone B (us-east-1b)
      ├── Public Subnet B:       10.100.2.0/24   -> ALB, NAT Gateway B
      ├── Private App Subnet B:  10.100.11.0/24  -> EC2 / ECS / EKS
      └── Private DB Subnet B:   10.100.21.0/24  -> RDS Aurora Standby
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
  description = "Deployment environment name"
  type        = string
  default     = "production"
}

variable "vpc_cidr" {
  description = "Base IPv4 CIDR block for the VPC"
  type        = string
  default     = "10.100.0.0/16"
}

variable "availability_zones" {
  description = "List of Availability Zones to deploy into"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.100.1.0/24", "10.100.2.0/24"]
}

variable "app_subnet_cidrs" {
  description = "CIDR blocks for private application subnets"
  type        = list(string)
  default     = ["10.100.10.0/24", "10.100.11.0/24"]
}

variable "db_subnet_cidrs" {
  description = "CIDR blocks for isolated private database subnets"
  type        = list(string)
  default     = ["10.100.20.0/24", "10.100.21.0/24"]
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Virtual Private Cloud (VPC)
# ------------------------------------------------------------------------------
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-core-vpc"
    Environment = var.environment
  }
}

# ------------------------------------------------------------------------------
# 2. Internet Gateway (IGW) for Public Ingress/Egress
# ------------------------------------------------------------------------------
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-igw"
  }
}

# ------------------------------------------------------------------------------
# 3. Subnets (Public, Private App, Private Database across 2 AZs)
# ------------------------------------------------------------------------------
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-subnet-${var.availability_zones[count.index]}"
    Tier = "Public"
  }
}

resource "aws_subnet" "app" {
  count                   = length(var.app_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.app_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name = "${var.environment}-app-subnet-${var.availability_zones[count.index]}"
    Tier = "Application"
  }
}

resource "aws_subnet" "db" {
  count                   = length(var.db_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.db_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name = "${var.environment}-db-subnet-${var.availability_zones[count.index]}"
    Tier = "Database"
  }
}

# ------------------------------------------------------------------------------
# 4. Elastic IPs and Multi-AZ NAT Gateways
# ------------------------------------------------------------------------------
resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"

  tags = {
    Name = "${var.environment}-nat-eip-${var.availability_zones[count.index]}"
  }
}

resource "aws_nat_gateway" "nat" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.environment}-nat-gw-${var.availability_zones[count.index]}"
  }

  depends_on = [aws_internet_gateway.igw]
}

# ------------------------------------------------------------------------------
# 5. Route Tables & Subnet Associations
# ------------------------------------------------------------------------------
# Public Route Table: Routes 0.0.0.0/0 to Internet Gateway
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.environment}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Application Route Tables: Routes 0.0.0.0/0 to respective NAT Gateway
resource "aws_route_table" "app" {
  count  = length(var.app_subnet_cidrs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat[count.index].id
  }

  tags = {
    Name = "${var.environment}-app-rt-${var.availability_zones[count.index]}"
  }
}

resource "aws_route_table_association" "app" {
  count          = length(aws_subnet.app)
  subnet_id      = aws_subnet.app[count.index].id
  route_table_id = aws_route_table.app[count.index].id
}

# Isolated Database Route Table: Local VPC route ONLY (No Internet Egress!)
resource "aws_route_table" "db" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-isolated-db-rt"
  }
}

resource "aws_route_table_association" "db" {
  count          = length(aws_subnet.db)
  subnet_id      = aws_subnet.db[count.index].id
  route_table_id = aws_route_table.db.id
}

# ------------------------------------------------------------------------------
# 6. S3 Gateway Endpoint (Free private connection to S3 for App Subnets)
# ------------------------------------------------------------------------------
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.app[*].id

  tags = {
    Name = "${var.environment}-s3-gateway-endpoint"
  }
}

# ------------------------------------------------------------------------------
# 7. VPC Flow Logs to CloudWatch Logs
# ------------------------------------------------------------------------------
resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/${var.environment}-flow-logs"
  retention_in_days = 30
}

resource "aws_iam_role" "flow_logs_role" {
  name_prefix = "vpc-flow-logs-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "vpc-flow-logs.amazonaws.com" }
      }
    ]
  })
}

resource "aws_iam_role_policy" "flow_logs_policy" {
  name_prefix = "vpc-flow-logs-policy-"
  role        = aws_iam_role.flow_logs_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Resource = "${aws_cloudwatch_log_group.flow_logs.arn}:*"
      }
    ]
  })
}

resource "aws_flow_log" "vpc_flow_log" {
  iam_role_arn    = aws_iam_role.flow_logs_role.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
}
```

### 1.3 `outputs.tf`
```hcl
output "vpc_id" {
  description = "The ID of the provisioned VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of Public Subnet IDs"
  value       = aws_subnet.public[*].id
}

output "app_subnet_ids" {
  description = "List of Private Application Subnet IDs"
  value       = aws_subnet.app[*].id
}

output "db_subnet_ids" {
  description = "List of Isolated Private Database Subnet IDs"
  value       = aws_subnet.db[*].id
}

output "nat_gateway_ips" {
  description = "Public Elastic IPs of the Multi-AZ NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create the VPC
1. Open the **Amazon VPC Console** (`https://console.aws.amazon.com/vpc/`).
2. In the top-right dashboard, click **Create VPC**.
3. Under **Resources to create**, select **VPC only**.
4. **Name tag:** `production-core-vpc`.
5. **IPv4 CIDR block:** Select **IPv4 CIDR manual input**, enter `10.100.0.0/16`.
6. **IPv6 CIDR block:** No IPv6 CIDR block.
7. **Tenancy:** Select **Default**.
8. Click **Create VPC**.
9. Select the newly created VPC, click **Actions** -> **Edit VPC settings**.
10. Check **Enable DNS hostnames** and ensure **Enable DNS resolution** is checked. Click **Save changes**.

---

### Step 2: Create and Attach the Internet Gateway (IGW)
1. In the VPC left navigation, click **Internet gateways**, then click **Create internet gateway**.
2. **Name tag:** `production-igw`. Click **Create internet gateway**.
3. With the IGW selected, click **Actions** -> **Attach to VPC**.
4. Select `production-core-vpc` and click **Attach internet gateway**.

---

### Step 3: Create the 6 Multi-Tier Subnets
Navigate to **Subnets** and click **Create subnet**. Select `production-core-vpc`:
* **Public Subnet 1:** Name: `production-public-subnet-us-east-1a`, AZ: `us-east-1a`, CIDR: `10.100.1.0/24`.
* **Public Subnet 2:** Name: `production-public-subnet-us-east-1b`, AZ: `us-east-1b`, CIDR: `10.100.2.0/24`.
* **App Subnet 1:** Name: `production-app-subnet-us-east-1a`, AZ: `us-east-1a`, CIDR: `10.100.10.0/24`.
* **App Subnet 2:** Name: `production-app-subnet-us-east-1b`, AZ: `us-east-1b`, CIDR: `10.100.11.0/24`.
* **DB Subnet 1:** Name: `production-db-subnet-us-east-1a`, AZ: `us-east-1a`, CIDR: `10.100.20.0/24`.
* **DB Subnet 2:** Name: `production-db-subnet-us-east-1b`, AZ: `us-east-1b`, CIDR: `10.100.21.0/24`.
Click **Create subnet**.

*Post-Creation Setting:* Select each Public Subnet -> **Actions** -> **Edit subnet settings** -> Check **Enable auto-assign public IPv4 address** -> **Save**.

---

### Step 4: Allocate Elastic IPs & Deploy Multi-AZ NAT Gateways
1. In the sidebar, click **NAT gateways** -> **Create NAT gateway**.
2. **NAT Gateway A:**
   * Name: `production-nat-gw-us-east-1a`.
   * Subnet: Select `production-public-subnet-us-east-1a`.
   * Connectivity type: **Public**.
   * Elastic IP allocation ID: Click **Allocate Elastic IP**.
   * Click **Create NAT gateway**.
3. **NAT Gateway B:**
   * Name: `production-nat-gw-us-east-1b`.
   * Subnet: Select `production-public-subnet-us-east-1b`.
   * Connectivity type: **Public**.
   * Elastic IP allocation ID: Click **Allocate Elastic IP**.
   * Click **Create NAT gateway**.

---

### Step 5: Configure Route Tables
1. **Public Route Table:**
   * Click **Route tables** -> **Create route table**. Name: `production-public-rt`, VPC: `production-core-vpc`.
   * Under **Routes** tab, click **Edit routes** -> **Add route**:
     * Destination: `0.0.0.0/0` | Target: **Internet Gateway** (`production-igw`). Click **Save changes**.
   * Under **Subnet associations** tab, associate both Public Subnets.
2. **Private App Route Table A:**
   * Create route table: `production-app-rt-us-east-1a`.
   * Add route: `0.0.0.0/0` -> Target: **NAT Gateway** (`production-nat-gw-us-east-1a`).
   * Associate: `production-app-subnet-us-east-1a`.
3. **Private App Route Table B:**
   * Create route table: `production-app-rt-us-east-1b`.
   * Add route: `0.0.0.0/0` -> Target: **NAT Gateway** (`production-nat-gw-us-east-1b`).
   * Associate: `production-app-subnet-us-east-1b`.
4. **Isolated Database Route Table:**
   * Create route table: `production-isolated-db-rt`.
   * **Do NOT add any default route `0.0.0.0/0`!**
   * Associate both Database subnets (`10.100.20.0/24` and `10.100.21.0/24`).

---

### Step 6: Verification Checks via AWS CLI
```bash
# Verify VPC route tables and route targets
aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=vpc-xxxxxx" \
    --query "RouteTables[*].{TableId:RouteTableId,Routes:Routes[*].{Dest:DestinationCidrBlock,Target:GatewayId}}" \
    --output table
```
