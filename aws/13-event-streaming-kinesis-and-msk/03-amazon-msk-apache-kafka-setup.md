# Amazon MSK (Apache Kafka) Cluster: Production Terraform & Console Setup

Amazon Managed Streaming for Apache Kafka (Amazon MSK) provides an enterprise, fully managed Apache Kafka service. In this guide, we engineer a **Multi-AZ Provisioned MSK Cluster** running across three Availability Zones with **AWS IAM Client Authentication**, in-transit TLS encryption, custom server configurations, and CloudWatch broker logging.

---

## Architecture Flow

```
[ Private VPC: 10.100.0.0/16 ]
  ├── Private Subnet AZ-1a ===> [ MSK Broker Node 1 (kafka.m5.large) ]
  ├── Private Subnet AZ-1b ===> [ MSK Broker Node 2 (kafka.m5.large) ]
  └── Private Subnet AZ-1c ===> [ MSK Broker Node 3 (kafka.m5.large) ]
             ^
             | Port 9098 (TLS + AWS IAM Client Authentication)
             |
[ Producer / Consumer EC2 Client ] (Assumes IAM Role with 'kafka-cluster:*' permissions)
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

variable "cluster_name" {
  description = "Name of the MSK Apache Kafka cluster"
  type        = string
  default     = "enterprise-kafka-prod"
}

variable "kafka_version" {
  description = "Apache Kafka engine version"
  type        = string
  default     = "3.5.1"
}

variable "broker_instance_type" {
  description = "EC2 instance family for Kafka brokers"
  type        = string
  default     = "kafka.m5.large"
}

variable "ebs_volume_size_gb" {
  description = "EBS storage per broker node in GB"
  type        = number
  default     = 100
}

variable "vpc_id" {
  description = "VPC ID where MSK brokers will be deployed"
  type        = string
}

variable "private_subnet_ids" {
  description = "List of exactly 3 private subnet IDs across 3 distinct AZs"
  type        = list(string)
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Custom Kafka Broker Configuration (server.properties)
# ------------------------------------------------------------------------------
resource "aws_msk_configuration" "kafka_config" {
  name              = "${var.cluster_name}-custom-config"
  kafka_versions    = [var.kafka_version]
  server_properties = <<EOF
auto.create.topics.enable = false
default.replication.factor = 3
min.insync.replicas = 2
num.partitions = 6
log.retention.hours = 72
EOF
}

# ------------------------------------------------------------------------------
# 2. CloudWatch Log Group for Kafka Broker Logs
# ------------------------------------------------------------------------------
resource "aws_cloudwatch_log_group" "msk_logs" {
  name              = "/aws/msk/${var.cluster_name}"
  retention_in_days = 30
}

# ------------------------------------------------------------------------------
# 3. Security Group for MSK Cluster
# ------------------------------------------------------------------------------
resource "aws_security_group" "msk_sg" {
  name        = "${var.cluster_name}-broker-sg"
  description = "Allow inbound Kafka traffic from authenticated VPC clients"
  vpc_id      = var.vpc_id

  # Port 9098: Secured TLS with AWS IAM Authentication
  ingress {
    description = "Kafka IAM TLS Client Port"
    from_port   = 9098
    to_port     = 9098
    protocol    = "tcp"
    cidr_blocks = ["10.100.0.0/16"] # Restricted to internal VPC CIDR
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.cluster_name}-broker-sg"
  }
}

# ------------------------------------------------------------------------------
# 4. Amazon MSK Apache Kafka Cluster
# ------------------------------------------------------------------------------
resource "aws_msk_cluster" "kafka_cluster" {
  cluster_name           = var.cluster_name
  kafka_version          = var.kafka_version
  number_of_broker_nodes = 3 # Exactly 1 broker per subnet across 3 AZs

  broker_node_group_info {
    instance_type   = var.broker_instance_type
    client_subnets  = var.private_subnet_ids
    security_groups = [aws_security_group.msk_sg.id]

    storage_info {
      ebs_storage_info {
        volume_size = var.ebs_volume_size_gb
      }
    }
  }

  configuration_info {
    arn      = aws_msk_configuration.kafka_config.arn
    revision = aws_msk_configuration.kafka_config.latest_revision
  }

  # In-Transit and At-Rest Encryption
  encryption_info {
    encryption_at_rest_kms_key_arn = "arn:aws:kms:${var.aws_region}:alias/aws/kafka"
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
  }

  # Authentication: Enforce IAM Authentication (No plaintext unauthenticated access)
  client_authentication {
    sasl {
      iam = true
    }
    unauthenticated = false
  }

  # Broker Logging
  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk_logs.name
      }
    }
  }

  tags = {
    Name = var.cluster_name
  }
}
```

### 1.3 `outputs.tf`
```hcl
output "cluster_arn" {
  description = "ARN of the Amazon MSK cluster"
  value       = aws_msk_cluster.kafka_cluster.arn
}

output "bootstrap_brokers_sasl_iam" {
  description = "Comma-separated list of Kafka broker endpoints for IAM authentication (Port 9098)"
  value       = aws_msk_cluster.kafka_cluster.bootstrap_brokers_sasl_iam
}

output "zookeeper_connect_string" {
  description = "Apache Zookeeper connection endpoints (if applicable)"
  value       = aws_msk_cluster.kafka_cluster.zookeeper_connect_string
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Stage 1: Network Prerequisites Verification
Before provisioning MSK, ensure you have:
1. A VPC with at least **3 Private Subnets**, each in a distinct Availability Zone (e.g., `us-east-1a`, `us-east-1b`, `us-east-1c`).
2. A Security Group allowing inbound TCP port **`9098`** from your application subnet CIDR.

---

### Stage 2: Create Custom MSK Configuration
1. Open the **Amazon MSK Console** (`https://console.aws.amazon.com/msk/`).
2. In the left navigation pane, click **Configurations**, then click **Create configuration**.
3. **Configuration name:** `enterprise-kafka-custom-config`.
4. **Kafka versions:** Select `3.5.1`.
5. In the **Configuration properties** text editor, enter:
   ```text
   auto.create.topics.enable = false
   default.replication.factor = 3
   min.insync.replicas = 2
   num.partitions = 6
   log.retention.hours = 72
   ```
6. Click **Create configuration**.

---

### Stage 3: Create the MSK Cluster
1. In the MSK navigation sidebar, click **Clusters**, then click **Create cluster**.
2. **Cluster creation method:** Choose **Custom create**.
3. **Cluster type:** Select **Provisioned** (or Serverless if zero cluster capacity management is required).
4. **Cluster name:** `enterprise-kafka-prod`.
5. **Apache Kafka version:** Choose `3.5.1`.
6. **Brokers:**
   * **Broker instance type:** Select `kafka.m5.large` (2 vCPU, 8 GiB RAM).
   * **Number of Zones:** Choose **3**.
   * **Brokers per zone:** Choose **1** (Total brokers = 3).
7. **Storage:**
   * **Storage type:** EBS storage.
   * **EBS storage volume per broker:** Enter `100` GiB.
8. **Configuration:**
   * Choose **Use a custom configuration**, and select `enterprise-kafka-custom-config`.
9. **Networking:**
   * **VPC:** Select your production VPC.
   * **Subnets:** Select your 3 private subnets across 3 AZs.
   * **Security groups:** Select your MSK security group allowing port 9098.
10. **Security & Authentication (Crucial Step):**
    * **Access control methods:**
      * **IAM role-based authentication:** **CHECKED / ENABLED**.
      * **SASL/SCRAM:** Optional.
      * **Unauthenticated access:** **UNCHECKED / DISABLED**.
    * **Encryption:**
      * **Between clients and brokers:** Select **TLS encryption only**.
      * **Within the cluster:** Select **Encrypted**.
      * **At rest:** Select AWS managed key (`aws/kafka`).
11. **Monitoring & Logging:**
    * **CloudWatch broker logs:** Check **Deliver to CloudWatch Logs**.
    * Enter log group: `/aws/msk/enterprise-kafka-prod`.
12. Review settings and click **Create cluster**.
    *(Note: MSK cluster provisioning deploys dedicated EC2 hardware across 3 AZs and takes approximately 15 to 25 minutes).*

---

### Stage 4: Client Authentication & Topic Creation Verification

From an EC2 instance in the same VPC equipped with an IAM Role granting `kafka-cluster:*` permissions:

#### 1. Download Kafka Client & AWS IAM Auth JAR:
```bash
# Download official Apache Kafka binaries
wget https://archive.apache.org/dist/kafka/3.5.1/kafka_2.13-3.5.1.tgz
tar -xzf kafka_2.13-3.5.1.tgz && cd kafka_2.13-3.5.1

# Download AWS MSK IAM Login Callback Handler JAR
wget https://github.com/aws/aws-msk-iam-auth/releases/download/v1.1.9/aws-msk-iam-auth-1.1.9-all.jar -P libs/
```

#### 2. Create `client.properties` for IAM Authentication:
```text
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler;
```

#### 3. Create a High-Availability Kafka Topic:
```bash
BOOTSTRAP_BROKERS="b-1.enterprise-kafka...:9098,b-2.enterprise-kafka...:9098,b-3.enterprise-kafka...:9098"

# Create topic with replication factor 3 across all 3 AZs
bin/kafka-topics.sh --create \
    --bootstrap-server "${BOOTSTRAP_BROKERS}" \
    --command-config client.properties \
    --replication-factor 3 \
    --partitions 6 \
    --topic payment-transactions
```

#### 4. Produce and Consume a Test Record:
```bash
# Run console producer
bin/kafka-console-producer.sh \
    --bootstrap-server "${BOOTSTRAP_BROKERS}" \
    --producer.config client.properties \
    --topic payment-transactions
> {"transaction_id":"tx-9821","amount":250.00,"currency":"USD"}

# In a separate terminal, run console consumer
bin/kafka-console-consumer.sh \
    --bootstrap-server "${BOOTSTRAP_BROKERS}" \
    --consumer.config client.properties \
    --topic payment-transactions \
    --from-beginning
```
