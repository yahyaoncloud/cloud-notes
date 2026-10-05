# Production EC2 Auto Scaling & Application Load Balancer: Terraform & Console Setup

This guide implements a scalable, resilient compute tier: an **Internet-Facing Application Load Balancer (ALB)** in public subnets distributing traffic across a **Target Tracking Auto Scaling Group (ASG)** of hardened EC2 instances running in private subnets with strict Security Group chaining and AWS Systems Manager (SSM) management.

---

## Architecture Flow

```
[ INTERNET ]
     |
     v (HTTPS Port 443)
[ ALB Security Group: Inbound 0.0.0.0/0 on Port 443 ]
[ Application Load Balancer ] (Across Public Subnets AZ-1a and AZ-1b)
     |
     v (HTTP Port 8080)
[ EC2 App Security Group: Inbound ONLY from ALB Security Group on Port 8080! ]
[ Auto Scaling Group (Min: 2, Desired: 2, Max: 10) ]
  ├── AZ-1a Private App Subnet: [ EC2 Instance #1 ] (No Public IP)
  └── AZ-1b Private App Subnet: [ EC2 Instance #2 ] (No Public IP)
```

---

## Part 1: Production Terraform Implementation

### 1.1 `variables.tf`
```hcl
variable "aws_region" {
  description = "AWS deployment region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment identifier"
  type        = string
  default     = "production"
}

variable "vpc_id" {
  description = "VPC ID where resources are deployed"
  type        = string
}

variable "public_subnet_ids" {
  description = "Public Subnet IDs for the Application Load Balancer"
  type        = list(string)
}

variable "app_subnet_ids" {
  description = "Private Subnet IDs for EC2 Auto Scaling instances"
  type        = list(string)
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t4g.medium" # ARM64 AWS Graviton (high performance, low cost)
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Security Groups (Chained Architecture)
# ------------------------------------------------------------------------------
# ALB Security Group
resource "aws_security_group" "alb_sg" {
  name        = "${var.environment}-alb-sg"
  description = "Allow inbound HTTPS from public internet"
  vpc_id      = var.vpc_id

  ingress {
    description = "Public HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Public HTTP for automated 301 redirect"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Outbound to private application tier"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-alb-sg" }
}

# EC2 App Security Group (Permits traffic ONLY from ALB SG)
resource "aws_security_group" "app_sg" {
  name        = "${var.environment}-app-ec2-sg"
  description = "Permit ingress ONLY from Application Load Balancer"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Inbound from ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id] # Security Group Chaining!
  }

  egress {
    description = "Outbound egress for packages via NAT Gateway"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-app-ec2-sg" }
}

# ------------------------------------------------------------------------------
# 2. Application Load Balancer & Target Group
# ------------------------------------------------------------------------------
resource "aws_lb" "external_alb" {
  name               = "${var.environment}-web-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = var.public_subnet_ids

  drop_invalid_header_fields = true
  enable_deletion_protection = false

  tags = { Name = "${var.environment}-web-alb" }
}

resource "aws_lb_target_group" "app_tg" {
  name        = "${var.environment}-app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "instance"

  deregistration_delay = 300 # 5-minute connection draining window

  health_check {
    enabled             = true
    path                = "/healthz"
    protocol            = "HTTP"
    port                = "8080"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200-299"
  }

  tags = { Name = "${var.environment}-app-tg" }
}

# HTTP Port 80 Listener (Redirects to HTTPS)
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.external_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ------------------------------------------------------------------------------
# 3. IAM Role for Systems Manager (SSM) - No SSH Keys Needed!
# ------------------------------------------------------------------------------
resource "aws_iam_role" "instance_role" {
  name_prefix = "ec2-ssm-instance-role-"

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

resource "aws_iam_role_policy_attachment" "ssm_core" {
  role       = aws_iam_role.instance_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "instance_profile" {
  name_prefix = "ec2-instance-profile-"
  role        = aws_iam_role.instance_role.name
}

# ------------------------------------------------------------------------------
# 4. EC2 Launch Template (IMDSv2 Enforced, GP3 Encrypted Storage)
# ------------------------------------------------------------------------------
data "aws_ami" "amazon_linux_2023_arm64" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-arm64"]
  }
}

resource "aws_launch_template" "asg_template" {
  name_prefix   = "${var.environment}-app-template-"
  image_id      = data.aws_ami.amazon_linux_2023_arm64.id
  instance_type = var.instance_type

  iam_instance_profile {
    arn = aws_iam_instance_profile.instance_profile.arn
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app_sg.id]
  }

  # Mandatory IMDSv2 configuration to prevent SSRF credential theft
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required" # Strict IMDSv2 token enforcement
    http_put_response_hop_limit = 1
  }

  # Modern GP3 Encrypted Root Volume
  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      iops                  = 3000
      throughput            = 125
      encrypted             = true
      delete_on_termination = true
    }
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    set -euo pipefail
    dnf update -y
    dnf install -y nginx
    
    # Configure NGINX to listen on port 8080 and serve health check
    cat <<'CONF' > /etc/nginx/conf.d/app.conf
    server {
        listen 8080;
        location /healthz {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
        location / {
            return 200 'Application Operational';
            add_header Content-Type text/plain;
        }
    }
    CONF
    
    systemctl enable nginx
    systemctl restart nginx
  EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.environment}-web-node"
      Environment = var.environment
    }
  }
}

# ------------------------------------------------------------------------------
# 5. Auto Scaling Group (ASG) & Target Tracking Policy
# ------------------------------------------------------------------------------
resource "aws_autoscaling_group" "app_asg" {
  name_prefix         = "${var.environment}-asg-"
  vpc_zone_identifier = var.app_subnet_ids
  target_group_arns   = [aws_lb_target_group.app_tg.arn]

  min_size     = 2
  desired_capacity = 2
  max_size     = 10

  health_check_type         = "ELB" # Uses ALB /healthz response rather than simple EC2 hypervisor check
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.asg_template.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
      instance_warmup        = 300
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Target Tracking Policy: Keep average ASG CPU at 60%
resource "aws_autoscaling_policy" "cpu_target_tracking" {
  name                   = "${var.environment}-cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app_asg.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}
```

### 1.3 `outputs.tf`
```hcl
output "alb_dns_name" {
  description = "The public DNS name of the Application Load Balancer"
  value       = aws_lb.external_alb.dns_name
}

output "asg_name" {
  description = "Name of the provisioned Auto Scaling Group"
  value       = aws_autoscaling_group.app_asg.name
}

output "target_group_arn" {
  description = "ARN of the ALB Target Group"
  value       = aws_lb_target_group.app_tg.arn
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create IAM Role for Systems Manager (SSM)
1. Open **IAM Console** -> **Roles** -> **Create role**.
2. **Trusted entity type:** Select **AWS service**, use case: **EC2**.
3. Under permissions policies, search for and select: **`AmazonSSMManagedInstanceCore`**.
4. Role name: `ec2-ssm-instance-role`. Click **Create role**.

---

### Step 2: Create Security Groups
1. **ALB Security Group:**
   * In the VPC Console, click **Security Groups** -> **Create security group**.
   * Name: `production-alb-sg`. VPC: `production-core-vpc`.
   * Add Inbound Rule: Type: **HTTP**, Port: `80`, Source: `0.0.0.0/0`.
   * Add Inbound Rule: Type: **HTTPS**, Port: `443`, Source: `0.0.0.0/0`. Click **Create security group**.
2. **EC2 App Security Group:**
   * Create security group: `production-app-ec2-sg`.
   * Add Inbound Rule: Type: **Custom TCP**, Port: `8080`.
   * **Source:** Click the search box, choose **Custom**, and select the Security Group ID of `production-alb-sg`. Click **Create security group**.

---

### Step 3: Create ALB Target Group & Application Load Balancer
1. In the EC2 Console left navigation, click **Target Groups** -> **Create target group**.
   * Target type: **Instances**.
   * Target group name: `production-app-tg`. Protocol: **HTTP**, Port: `8080`.
   * Health checks: Path: `/healthz`. Advanced settings: Healthy threshold: `2`, Unhealthy: `3`, Timeout: `5s`, Interval: `30s`. Click **Create target group**.
2. In the navigation, click **Load Balancers** -> **Create load balancer** -> **Application Load Balancer**.
   * Name: `production-web-alb`. Scheme: **Internet-facing**. IP address type: **IPv4**.
   * Network mapping: Select `production-core-vpc` and check both **Public Subnets**.
   * Security groups: Select `production-alb-sg`.
   * Listeners: Port `80` forwarding to `production-app-tg` (or configure redirect to HTTPS). Click **Create load balancer**.

---

### Step 4: Create Launch Template
1. In EC2 Console, click **Launch Templates** -> **Create launch template**.
2. Name: `production-app-template`.
3. AMI: Select **Amazon Linux 2023** (Architecture: **64-bit (ARM)** if using t4g).
4. Instance type: `t4g.medium`.
5. Key pair: Select **Proceed without a key pair** (we use SSM Session Manager!).
6. Network settings: Security group: `production-app-ec2-sg`. Auto-assign public IP: **Disable**.
7. Storage: Select `gp3`, 30 GiB, Encrypted.
8. Expand **Advanced details**:
   * IAM instance profile: Select `ec2-ssm-instance-role`.
   * Metadata accessible: **Enabled**.
   * Metadata version: Select **V2 only (token required)**.
   * User data: Paste the NGINX bootstrap script.
9. Click **Create launch template**.

---

### Step 5: Create Auto Scaling Group (ASG)
1. Click **Auto Scaling Groups** -> **Create Auto Scaling group**.
2. Name: `production-asg`. Select launch template `production-app-template`.
3. Network: Select `production-core-vpc` and pick both **Private Application Subnets**.
4. Load balancing: Select **Attach to an existing load balancer**, and choose `production-app-tg`.
5. Health checks: Check **Turn on Elastic Load Balancing health checks**. Grace period: `300` seconds.
6. Group size: Min: `2`, Desired: `2`, Max: `10`.
7. Scaling policies: Select **Target tracking scaling policy**. Metric: **Average CPU utilization**, Target value: `60`.
8. Click through and click **Create Auto Scaling group**.

---

### Step 6: Verification
Test end-to-end load-balanced response:
```bash
# Query the ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers --names production-web-alb --query "LoadBalancers[0].DNSName" --output text)
curl -i "http://${ALB_DNS}/healthz"
# Returns: HTTP/1.1 200 OK - 'OK'
```
