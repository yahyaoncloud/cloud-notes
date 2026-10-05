# Production Amazon Aurora PostgreSQL Multi-AZ & RDS Proxy: Terraform & Console Setup

Amazon Aurora PostgreSQL delivers high-throughput transactional database performance with automated multi-zone replication across 3 Availability Zones. This guide implements an enterprise **Aurora Serverless v2 / Multi-AZ Cluster** situated in isolated private database subnets, secured via **AWS Secrets Manager** password management and integrated with an **Amazon RDS Proxy** for connection multiplexing.

---

## Architecture Flow

```
[ ECS / EC2 Application Tier ]
             |
             v (PostgreSQL Port 5432)
[ Amazon RDS Proxy Endpoint ] (Manages shared warm pool of DB connections)
             |
             +-----------------------+
             | (Writer Endpoint)     | (Reader Endpoint)
             v                       v
[ Aurora Primary DB Instance ]  [ Aurora Reader Replica Instance ]
  (AZ-1a Private DB Subnet)       (AZ-1b Private DB Subnet)
             \                       /
   <========== Shared Aurora 6-Way Distributed Storage ==========>
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

variable "vpc_id" {
  description = "VPC ID where database is deployed"
  type        = string
}

variable "db_subnet_ids" {
  description = "List of isolated private database subnet IDs"
  type        = list(string)
}

variable "app_security_group_id" {
  description = "Security Group ID of the compute application tier"
  type        = string
}

variable "database_name" {
  description = "Name of the default PostgreSQL database"
  type        = string
  default     = "enterprisedb"
}

variable "min_capacity" {
  description = "Minimum Aurora Serverless v2 Capacity Units (ACUs)"
  type        = number
  default     = 0.5
}

variable "max_capacity" {
  description = "Maximum Aurora Serverless v2 Capacity Units (ACUs)"
  type        = number
  default     = 16.0
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Database Subnet Group (Isolated Subnets)
# ------------------------------------------------------------------------------
resource "aws_db_subnet_group" "aurora_subnet_group" {
  name       = "${var.environment}-aurora-subnet-group"
  subnet_ids = var.db_subnet_ids

  tags = { Name = "${var.environment}-aurora-subnet-group" }
}

# ------------------------------------------------------------------------------
# 2. Database Security Group (Chained to Application SG)
# ------------------------------------------------------------------------------
resource "aws_security_group" "db_sg" {
  name        = "${var.environment}-aurora-db-sg"
  description = "Allow inbound PostgreSQL traffic ONLY from application tier"
  vpc_id      = var.vpc_id

  ingress {
    description     = "PostgreSQL Ingress from App SG"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id] # Security Group Chaining!
  }

  egress {
    description = "Allow outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-aurora-db-sg" }
}

# ------------------------------------------------------------------------------
# 3. AWS Secrets Manager Master Password
# ------------------------------------------------------------------------------
resource "random_password" "master_password" {
  length           = 24
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_secretsmanager_secret" "db_secret" {
  name_prefix = "${var.environment}-aurora-master-creds-"
}

resource "aws_secretsmanager_secret_version" "db_secret_val" {
  secret_id = aws_secretsmanager_secret.db_secret.id
  secret_string = jsonencode({
    engine   = "postgres"
    host     = aws_rds_cluster.aurora_cluster.endpoint
    port     = 5432
    username = "postgresadmin"
    password = random_password.master_password.result
    database = var.database_name
  })
}

# ------------------------------------------------------------------------------
# 4. Amazon Aurora PostgreSQL Cluster (Serverless v2)
# ------------------------------------------------------------------------------
resource "aws_rds_cluster" "aurora_cluster" {
  cluster_identifier = "${var.environment}-aurora-pg-cluster"
  engine             = "aurora-postgresql"
  engine_mode        = "provisioned"
  engine_version     = "15.4"
  database_name      = var.database_name
  master_username    = "postgresadmin"
  master_password    = random_password.master_password.result

  db_subnet_group_name   = aws_db_subnet_group.aurora_subnet_group.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]

  storage_encrypted = true
  deletion_protection = false # Set true for strict production workloads
  skip_final_snapshot = true

  serverlessv2_scaling_configuration {
    min_capacity = var.min_capacity
    max_capacity = var.max_capacity
  }

  tags = { Name = "${var.environment}-aurora-pg-cluster" }
}

# Primary Writer Instance
resource "aws_rds_cluster_instance" "writer" {
  cluster_identifier = aws_rds_cluster.aurora_cluster.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora_cluster.engine
  engine_version     = aws_rds_cluster.aurora_cluster.engine_version
  identifier         = "${var.environment}-aurora-instance-1"
  publicly_accessible = false
}

# Secondary Reader Replica (Multi-AZ Failover Target)
resource "aws_rds_cluster_instance" "reader" {
  cluster_identifier = aws_rds_cluster.aurora_cluster.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora_cluster.engine
  engine_version     = aws_rds_cluster.aurora_cluster.engine_version
  identifier         = "${var.environment}-aurora-instance-2"
  publicly_accessible = false
}

# ------------------------------------------------------------------------------
# 5. Amazon RDS Proxy (Connection Pooling)
# ------------------------------------------------------------------------------
resource "aws_iam_role" "proxy_role" {
  name_prefix = "rds-proxy-secrets-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "rds.amazonaws.com" }
      }
    ]
  })
}

resource "aws_iam_role_policy" "proxy_secrets_policy" {
  name_prefix = "rds-proxy-secrets-policy-"
  role        = aws_iam_role.proxy_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["secretsmanager:GetSecretValue"]
        Resource = aws_secretsmanager_secret.db_secret.arn
      }
    ]
  })
}

resource "aws_db_proxy" "aurora_proxy" {
  name                   = "${var.environment}-db-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.proxy_role.arn
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  vpc_subnet_ids         = var.db_subnet_ids

  auth {
    auth_scheme = "SECRETS"
    secret_arn  = aws_secretsmanager_secret.db_secret.arn
    iam_auth    = "DISABLED"
  }

  tags = { Name = "${var.environment}-db-proxy" }
}

resource "aws_db_proxy_default_target_group" "proxy_target_group" {
  db_proxy_name = aws_db_proxy.aurora_proxy.name

  connection_pool_config {
    max_connections_percent      = 90
    max_idle_connections_percent = 50
  }
}

resource "aws_db_proxy_target" "aurora_target" {
  db_proxy_name         = aws_db_proxy.aurora_proxy.name
  target_group_name     = aws_db_proxy_default_target_group.proxy_target_group.name
  db_cluster_identifier = aws_rds_cluster.aurora_cluster.id
}
```

### 1.3 `outputs.tf`
```hcl
output "aurora_writer_endpoint" {
  description = "The primary writer cluster endpoint"
  value       = aws_rds_cluster.aurora_cluster.endpoint
}

output "aurora_reader_endpoint" {
  description = "The read-only load-balanced replica endpoint"
  value       = aws_rds_cluster.aurora_cluster.reader_endpoint
}

output "rds_proxy_endpoint" {
  description = "The connection pooling proxy endpoint"
  value       = aws_db_proxy.aurora_proxy.endpoint
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create DB Subnet Group
1. Open **Amazon RDS Console** -> **Subnet groups** -> **Create DB subnet group**.
2. Name: `production-aurora-subnet-group`.
3. VPC: `production-core-vpc`.
4. Availability Zones: Select `us-east-1a` and `us-east-1b`.
5. Subnets: Choose both **Private Database Subnets** (`10.100.20.0/24` and `10.100.21.0/24`). Click **Create**.

---

### Step 2: Create Aurora PostgreSQL Cluster
1. In RDS Console, click **Databases** -> **Create database**.
2. Choose database creation method: **Standard create**.
3. Engine type: **Amazon Aurora**. Edition: **PostgreSQL-Compatible**.
4. Engine version: `Aurora PostgreSQL 15.4`.
5. Templates: Select **Production**.
6. **Settings:**
   * DB cluster identifier: `production-aurora-pg-cluster`.
   * Master username: `postgresadmin`.
   * Credentials management: Choose **Auto generate a password** and check **Manage in AWS Secrets Manager**.
7. **Cluster storage configuration:** Select **Aurora Standard**.
8. **Instance configuration:**
   * Choose **Serverless v2** (or Provisioned `db.r6g.large`).
   * Set Min ACUs: `0.5`, Max ACUs: `16`.
9. **Availability & durability:**
   * Select **Create an Aurora Replica / Reader node in a different DB Instance (Multi-AZ)**.
10. **Connectivity:**
    * VPC: `production-core-vpc`.
    * DB Subnet group: `production-aurora-subnet-group`.
    * Public access: **No**.
    * VPC security group: Select `production-aurora-db-sg`.
11. **Database authentication:** Password authentication. Click **Create database**.

---

### Step 3: Create Amazon RDS Proxy
1. In RDS Console sidebar, click **Proxies** -> **Create proxy**.
2. Proxy identifier: `production-db-proxy`. Engine family: **PostgreSQL**.
3. Target database: Select `production-aurora-pg-cluster`.
4. Authentication: Select the AWS Secrets Manager secret generated by RDS.
5. IAM role: Choose **Create IAM role**.
6. Connectivity:
   * Subnets: Select the Database Subnets.
   * Security group: `production-aurora-db-sg`.
7. Click **Create proxy**.
