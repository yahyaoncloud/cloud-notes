# Production Amazon ECS Fargate Container Service: Terraform & Console Setup

AWS Fargate provides serverless compute for containerized microservices, eliminating the need to provision, patch, or scale underlying EC2 instances. This guide implements a production-grade **Amazon ECS Fargate Service** registered behind an Application Load Balancer with strict IAM role separation and CloudWatch logging.

---

## Architecture Flow

```
[ Application Load Balancer ]
             |
             v (HTTP Port 8080)
[ ECS Fargate Service: "payment-api-service" ] (Desired Count: 2)
  ├── Task Execution Role: Pulls image from ECR + streams logs to CloudWatch
  ├── Task Role: Grants container permissions to access Amazon S3 / DynamoDB
  └── awsvpc Networking: Each task gets a dedicated private ENI in App Subnets
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
  description = "VPC ID where ECS tasks will run"
  type        = string
}

variable "app_subnet_ids" {
  description = "Private Subnet IDs for Fargate tasks"
  type        = list(string)
}

variable "alb_security_group_id" {
  description = "Security Group ID of the Application Load Balancer"
  type        = string
}

variable "alb_target_group_arn" {
  description = "ARN of the ALB Target Group for ECS traffic"
  type        = string
}

variable "container_image" {
  description = "Container image URI (ECR or public image)"
  type        = string
  default     = "public.ecr.aws/nginx/nginx:alpine"
}

variable "container_cpu" {
  description = "vCPU reservation (256 = 0.25 vCPU, 512 = 0.5 vCPU, 1024 = 1 vCPU)"
  type        = number
  default     = 256
}

variable "container_memory" {
  description = "Memory reservation in MB (512, 1024, 2048)"
  type        = number
  default     = 512
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Amazon ECR Repository for Microservice Images
# ------------------------------------------------------------------------------
resource "aws_ecr_repository" "app_repo" {
  name                 = "${var.environment}-payment-api"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
  }
}

# ------------------------------------------------------------------------------
# 2. Amazon ECS Cluster
# ------------------------------------------------------------------------------
resource "aws_ecs_cluster" "ecs_cluster" {
  name = "${var.environment}-ecs-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_cluster_capacity_providers" "cluster_capacity" {
  cluster_name = aws_ecs_cluster.ecs_cluster.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }
}

# ------------------------------------------------------------------------------
# 3. CloudWatch Log Group for Container Output
# ------------------------------------------------------------------------------
resource "aws_cloudwatch_log_group" "ecs_logs" {
  name              = "/ecs/${var.environment}-payment-api"
  retention_in_days = 30
}

# ------------------------------------------------------------------------------
# 4. IAM Roles: Task Execution Role vs. Task Role
# ------------------------------------------------------------------------------
# A. Task Execution Role (Used by AWS agent to pull ECR images and send logs)
resource "aws_iam_role" "ecs_execution_role" {
  name_prefix = "ecs-execution-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "ecs-tasks.amazonaws.com" }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution_policy" {
  role       = aws_iam_role.ecs_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# B. Task Role (Used by the APPLICATION inside the container to access AWS services)
resource "aws_iam_role" "ecs_task_role" {
  name_prefix = "ecs-app-task-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "ecs-tasks.amazonaws.com" }
      }
    ]
  })
}

# Example: Grant application permission to access an S3 bucket
resource "aws_iam_role_policy" "task_s3_policy" {
  name_prefix = "ecs-app-s3-access-"
  role        = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::*/*"
      }
    ]
  })
}

# ------------------------------------------------------------------------------
# 5. ECS Security Group (Chained to ALB)
# ------------------------------------------------------------------------------
resource "aws_security_group" "ecs_tasks_sg" {
  name        = "${var.environment}-ecs-tasks-sg"
  description = "Permit ingress ONLY from Application Load Balancer on container port"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Inbound from ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [var.alb_security_group_id]
  }

  egress {
    description = "Allow all outbound traffic for image pulls and API calls"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-ecs-tasks-sg" }
}

# ------------------------------------------------------------------------------
# 6. ECS Task Definition
# ------------------------------------------------------------------------------
resource "aws_ecs_task_definition" "app_task" {
  family                   = "${var.environment}-payment-api"
  network_mode             = "awsvpc" # Mandatory for AWS Fargate
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.container_cpu
  memory                   = var.container_memory

  execution_role_arn = aws_iam_role.ecs_execution_role.arn
  task_role_arn      = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name      = "payment-api-container"
      image     = var.container_image
      essential = true

      portMappings = [
        {
          containerPort = 8080
          hostPort      = 8080
          protocol      = "tcp"
        }
      ]

      environment = [
        { name = "ENVIRONMENT", value = var.environment },
        { name = "PORT", value = "8080" }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.ecs_logs.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "payment-api"
        }
      }
    }
  ])
}

# ------------------------------------------------------------------------------
# 7. ECS Fargate Service
# ------------------------------------------------------------------------------
resource "aws_ecs_service" "app_service" {
  name            = "${var.environment}-payment-service"
  cluster         = aws_ecs_cluster.ecs_cluster.id
  task_definition = aws_ecs_task_definition.app_task.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  deployment_maximum_percent         = 200 # Spawns 2 new tasks before terminating old
  deployment_minimum_healthy_percent = 100 # Never drops below 2 healthy tasks

  network_configuration {
    subnets          = var.app_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks_sg.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = var.alb_target_group_arn
    container_name   = "payment-api-container"
    container_port   = 8080
  }

  depends_on = [aws_iam_role_policy_attachment.ecs_execution_policy]
}
```

### 1.3 `outputs.tf`
```hcl
output "cluster_name" {
  description = "Name of the ECS cluster"
  value       = aws_ecs_cluster.ecs_cluster.name
}

output "service_name" {
  description = "Name of the ECS service"
  value       = aws_ecs_service.app_service.name
}

output "ecr_repository_url" {
  description = "URL of the ECR repository"
  value       = aws_ecr_repository.app_repo.repository_url
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create Amazon ECR Repository
1. Open **Amazon ECR Console** -> **Repositories** -> **Create repository**.
2. Visibility settings: **Private**. Repository name: `production-payment-api`.
3. Tag immutability: **Enable** (recommended for production).
4. Scan on push: **Enable**. Encryption: **KMS encryption**.
5. Click **Create repository**.

---

### Step 2: Create the ECS Cluster
1. Open **Amazon ECS Console** -> **Clusters** -> **Create cluster**.
2. Cluster name: `production-ecs-cluster`.
3. Infrastructure: Select **AWS Fargate (serverless)**.
4. Monitoring: Check **Use Container Insights**.
5. Click **Create**.

---

### Step 3: Create Task Definition
1. In ECS Console left navigation, click **Task definitions** -> **Create new task definition** -> **Create new task definition with JSON** (or UI builder).
2. Task definition family: `production-payment-api`.
3. Launch type: **AWS Fargate**.
4. Operating system: **Linux/X86_64** (or ARM64).
5. CPU: `0.25 vCPU (256)`. Memory: `0.5 GB (512)`.
6. **Task execution role:** Choose **Create new role** or `ecsTaskExecutionRole`.
7. **Task role:** Select your custom application role (e.g. `ecs-app-task-role`).
8. **Container details:**
   * Container name: `payment-api-container`.
   * Image URI: Paste your ECR or public image URL.
   * Container port: `8080`, protocol: `TCP`, app protocol: `HTTP`.
   * Logging: Select **Amazon CloudWatch logs** (auto-generates log group).
9. Click **Create**.

---

### Step 4: Deploy ECS Fargate Service behind ALB
1. Navigate back to your cluster `production-ecs-cluster`.
2. Under the **Services** tab, click **Create**.
3. **Environment:**
   * Compute configuration: **Launch type**. Launch type: **FARGATE**.
4. **Deployment configuration:**
   * Application type: **Service**.
   * Family: `production-payment-api`, Revision: `Latest`.
   * Service name: `production-payment-service`.
   * Desired tasks: `2`.
5. **Networking:**
   * VPC: `production-core-vpc`.
   * Subnets: Select both **Private Application Subnets**.
   * Security group: Create or select `production-ecs-tasks-sg` (permitting Port 8080 from ALB SG).
   * Public IP: **Turned OFF / Disabled**.
6. **Load balancing:**
   * Load balancer type: **Application Load Balancer**.
   * Select your existing ALB.
   * Target group: Select your existing target group (Port 8080).
7. Click **Create**.

---

### Step 5: Verification
```bash
# Check service stability and running task status
aws ecs describe-services \
    --cluster production-ecs-cluster \
    --services production-payment-service \
    --query "services[0].{DesiredCount:desiredCount,RunningCount:runningCount,Status:status}" \
    --output table
```
