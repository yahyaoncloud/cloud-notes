# Amazon Kinesis Data Streams & Firehose: Production Terraform & Console Setup

This guide provides an end-to-end, production-grade setup for an enterprise real-time streaming pipeline: **Amazon Kinesis Data Streams** ingesting events, replicating to **Amazon Data Firehose**, and loading compressed JSON data into **Amazon S3** with CloudWatch monitoring.

It contains two complete implementation paths:
1. **Infrastructure as Code (Terraform):** Modular, production-ready HCL with `main.tf`, `variables.tf`, `outputs.tf`, and `providers.tf`.
2. **AWS Management Console Walkthrough:** Exact step-by-step clickthrough instructions with security configurations and verification tests.

---

## Architecture Flow

```
[ Event Producers ]
        |
        v (PutRecord / PutRecords API)
[ Amazon Kinesis Data Stream ] (Provisioned or On-Demand)
        |
        v (Assumes Firehose Delivery IAM Role)
[ Amazon Data Firehose Delivery Stream ]
        |
        |-- Buffering: 300 seconds OR 5 MB (Whichever occurs first)
        |-- GZIP Compression enabled
        |-- CloudWatch Logs error monitoring
        v
[ Amazon S3 Analytics Data Lake ] (`s3://streaming-data-lake-prod/raw/YYYY/MM/DD/`)
```

---

## Part 1: Production Terraform Implementation

### 1.1 `providers.tf`
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = "EventStreaming"
    }
  }
}
```

### 1.2 `variables.tf`
```hcl
variable "aws_region" {
  description = "Target AWS Region for deployment"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment name"
  type        = string
  default     = "production"
}

variable "stream_name" {
  description = "Name of the Kinesis Data Stream"
  type        = string
  default     = "telemetry-event-stream"
}

variable "stream_mode" {
  description = "Capacity mode: ON_DEMAND or PROVISIONED"
  type        = string
  default     = "ON_DEMAND"
}

variable "retention_period_hours" {
  description = "Data record retention period in hours (24 to 8760)"
  type        = number
  default     = 48
}

variable "buffer_size_mb" {
  description = "Firehose delivery buffer size in MB (1 to 128)"
  type        = number
  default     = 5
}

variable "buffer_interval_seconds" {
  description = "Firehose delivery buffer time interval in seconds (60 to 900)"
  type        = number
  default     = 300
}
```

### 1.3 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Amazon S3 Destination Bucket for Streaming Archives
# ------------------------------------------------------------------------------
resource "aws_s3_bucket" "streaming_bucket" {
  bucket_prefix = "streaming-data-lake-${var.environment}-"
  force_destroy = false
}

resource "aws_s3_bucket_versioning" "bucket_versioning" {
  bucket = aws_s3_bucket.streaming_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "bucket_crypto" {
  bucket = aws_s3_bucket.streaming_bucket.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket                  = aws_s3_bucket.streaming_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ------------------------------------------------------------------------------
# 2. Amazon Kinesis Data Stream (Ingestion Tier)
# ------------------------------------------------------------------------------
resource "aws_kinesis_stream" "ingest_stream" {
  name             = var.stream_name
  retention_period = var.retention_period_hours

  stream_mode_details {
    stream_mode = var.stream_mode
  }

  encryption_type = "KMS"
  kms_key_id      = "alias/aws/kinesis"

  tags = {
    Name = var.stream_name
  }
}

# ------------------------------------------------------------------------------
# 3. CloudWatch Log Group for Firehose Telemetry
# ------------------------------------------------------------------------------
resource "aws_cloudwatch_log_group" "firehose_log_group" {
  name              = "/aws/kinesisfirehose/${var.stream_name}-delivery"
  retention_in_days = 30
}

resource "aws_cloudwatch_log_stream" "firehose_log_stream" {
  name           = "S3Delivery"
  log_group_name = aws_cloudwatch_log_group.firehose_log_group.name
}

# ------------------------------------------------------------------------------
# 4. IAM Role & Least-Privilege Policies for Firehose
# ------------------------------------------------------------------------------
resource "aws_iam_role" "firehose_role" {
  name_prefix = "firehose-delivery-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "firehose.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "firehose_permissions" {
  name_prefix = "firehose-delivery-policy-"
  role        = aws_iam_role.firehose_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Read permissions on Kinesis Data Stream
      {
        Effect = "Allow"
        Action = [
          "kinesis:DescribeStream",
          "kinesis:GetShardIterator",
          "kinesis:GetRecords",
          "kinesis:ListShards"
        ]
        Resource = aws_kinesis_stream.ingest_stream.arn
      },
      # Write permissions to S3 Destination Bucket
      {
        Effect = "Allow"
        Action = [
          "s3:AbortMultipartUpload",
          "s3:GetBucketLocation",
          "s3:GetObject",
          "s3:ListBucket",
          "s3:ListBucketMultipartUploads",
          "s3:PutObject"
        ]
        Resource = [
          aws_s3_bucket.streaming_bucket.arn,
          "${aws_s3_bucket.streaming_bucket.arn}/*"
        ]
      },
      # Log streaming permissions to CloudWatch
      {
        Effect = "Allow"
        Action = [
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.firehose_log_group.arn}:*"
      }
    ]
  })
}

# ------------------------------------------------------------------------------
# 5. Amazon Data Firehose Delivery Stream
# ------------------------------------------------------------------------------
resource "aws_kinesis_firehose_delivery_stream" "s3_stream" {
  name        = "${var.stream_name}-firehose-s3"
  destination = "extended_s3"

  kinesis_source_configuration {
    kinesis_stream_arn = aws_kinesis_stream.ingest_stream.arn
    role_arn           = aws_iam_role.firehose_role.arn
  }

  extended_s3_configuration {
    role_arn   = aws_iam_role.firehose_role.arn
    bucket_arn = aws_s3_bucket.streaming_bucket.arn

    buffering_size     = var.buffer_size_mb
    buffering_interval = var.buffer_interval_seconds
    compression_format = "GZIP"

    prefix              = "raw/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"
    error_output_prefix = "errors/year=!{timestamp:yyyy}/month=!{timestamp:MM}/!{firehose:error-output-type}/"

    cloudwatch_logging_options {
      enabled         = true
      log_group_name  = aws_cloudwatch_log_group.firehose_log_group.name
      log_stream_name = aws_cloudwatch_log_stream.firehose_log_stream.name
    }
  }
}
```

### 1.4 `outputs.tf`
```hcl
output "kinesis_stream_arn" {
  description = "ARN of the primary Kinesis Data Stream"
  value       = aws_kinesis_stream.ingest_stream.arn
}

output "kinesis_stream_name" {
  description = "Name of the Kinesis Data Stream"
  value       = aws_kinesis_stream.ingest_stream.name
}

output "firehose_delivery_arn" {
  description = "ARN of the Firehose Delivery Stream"
  value       = aws_kinesis_firehose_delivery_stream.s3_stream.arn
}

output "s3_destination_bucket" {
  description = "S3 bucket storing streaming archives"
  value       = aws_s3_bucket.streaming_bucket.bucket
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

If you are setting this up via the AWS Web Console or defending the setup steps during an interview, follow this exact click-by-click procedure:

### Stage 1: Create Destination S3 Bucket
1. Open the **Amazon S3 Console** (`https://console.aws.amazon.com/s3/`).
2. Click **Create bucket**.
3. **Bucket name:** Enter a globally unique name, e.g., `streaming-data-lake-prod-94821`.
4. **AWS Region:** Choose `US East (N. Virginia) us-east-1`.
5. **Block Public Access settings:** Ensure **Block all public access** remains **CHECKED** (all 4 checkboxes active).
6. **Bucket Versioning:** Select **Enable**.
7. **Default encryption:** Select **Server-side encryption with Amazon S3 managed keys (SSE-S3)** with Bucket Key enabled.
8. Click **Create bucket**.

---

### Stage 2: Create the Kinesis Data Stream
1. Open the **Amazon Kinesis Console** (`https://console.aws.amazon.com/kinesis/`).
2. In the navigation sidebar, select **Data streams**, then click **Create data stream**.
3. **Data stream name:** Enter `telemetry-event-stream`.
4. **Capacity mode:**
   * Select **On-demand** (recommended for variable or unknown production workloads).
   * *(Note: If selecting Provisioned, specify 2 Shards to support 2 MB/sec write, 4 MB/sec read).*
5. Expand **Configuration settings**:
   * **Data retention period:** Change from 24 hours to `48 hours`.
   * **Server-side encryption:** Enable with `aws/kinesis` AWS Managed KMS key.
6. Click **Create data stream**. Wait ~15 seconds until status transitions to **Active**.

---

### Stage 3: Create Amazon Data Firehose Delivery Stream
1. In the Kinesis console sidebar, click **Delivery streams** (or search for **Amazon Data Firehose**).
2. Click **Create delivery stream**.
3. **Source and destination:**
   * **Source:** Select **Amazon Kinesis Data Streams**.
   * **Destination:** Select **Amazon S3**.
4. **Source settings:**
   * Click **Browse** and select your active stream `telemetry-event-stream`.
5. **Delivery stream name:** Enter `telemetry-event-stream-firehose-s3`.
6. **Destination settings (Amazon S3):**
   * **S3 bucket:** Click **Browse** and pick the bucket created in Stage 1 (`streaming-data-lake-prod-94821`).
   * **Dynamic partitioning:** Disabled (unless routing by payload keys).
   * **S3 bucket prefix:** Enter: `raw/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/`
   * **S3 bucket error output prefix:** Enter: `errors/year=!{timestamp:yyyy}/month=!{timestamp:MM}/!{firehose:error-output-type}/`
7. **Buffer hints, compression and encryption:**
   * **Buffer size:** Enter `5` (MB).
   * **Buffer interval:** Enter `300` (Seconds).
   * **Compression for data records:** Select **GZIP**.
   * **Encryption:** Keep disabled or select KMS.
8. **Advanced settings:**
   * **CloudWatch error logging:** Select **Turn on**.
   * **Permissions:** Select **Create or update IAM role `KinesisFirehoseServiceRole-...`** (AWS will automatically attach the correct trust policy and resource ARNs).
9. Click **Create delivery stream**.

---

### Stage 4: Testing & Live Verification

Verify the end-to-end pipeline using the AWS CLI:

#### 1. Ingest Test Records into Kinesis Data Stream:
```bash
# Push a test telemetry record into the stream
aws kinesis put-record \
    --stream-name telemetry-event-stream \
    --partition-key "sensor-device-001" \
    --data '{"device_id":"sensor-001","temperature":72.5,"status":"NORMAL"}' \
    --region us-east-1
```

#### 2. Confirm Delivery in Amazon S3:
Wait 5 minutes (or push 5 MB of data to trigger the buffer threshold immediately):
```bash
# List objects generated by Firehose in S3
aws s3 ls s3://streaming-data-lake-prod-94821/raw/ --recursive --human-readable
```

#### 3. Inspect Ingested Payload:
```bash
# Download and decompress the GZIP archive
aws s3 cp s3://streaming-data-lake-prod-94821/raw/year=2026/.../telemetry-file.gz .
gzip -d telemetry-file.gz
cat telemetry-file
# Output: {"device_id":"sensor-001","temperature":72.5,"status":"NORMAL"}
```
