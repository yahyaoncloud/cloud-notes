# Enterprise Amazon S3 Security & Lifecycle Setup: Terraform & Console Guide

Amazon S3 is the primary target for data lakes, database backups, and media assets. In production environments, misconfigured S3 buckets represent the most common cause of cloud data breaches. This guide implements an enterprise-hardened S3 bucket protected with a **Customer Managed KMS Key (CMK)**, **Strict Block Public Access**, **TLS-only enforcement**, and automated **Lifecycle transition tiers** down to Glacier Deep Archive.

---

## Architecture Flow

```
[ Incoming S3 Upload / Request ]
               |
               v
+-------------------------------------------------------------+
| 1. S3 Block Public Access (Enforces all 4 public block flags)|
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
| 2. Bucket Policy:                                           |
|    - DENY any request without TLS encryption in transit     |
|    - DENY any upload not encrypted with dedicated KMS CMK   |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
| 3. Customer Managed KMS Key (CMK)                           |
|    - Automatic 365-day cryptographic key rotation           |
|    - S3 Bucket Key enabled (Reduces KMS API cost by 99%)    |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
| 4. Lifecycle Tiering Engine:                                |
|    - Day 0: S3 Standard                                     |
|    - Day 30: Transition to S3 Standard-IA                   |
|    - Day 90: Transition to Glacier Flexible Retrieval       |
|    - Day 180: Transition to Glacier Deep Archive ($1/TB/mo) |
+-------------------------------------------------------------+
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

variable "bucket_name_prefix" {
  description = "Prefix for the S3 bucket name"
  type        = string
  default     = "enterprise-secure-lake"
}
```

### 1.2 `main.tf`
```hcl
data "aws_caller_identity" "current" {}

# ------------------------------------------------------------------------------
# 1. Customer Managed KMS Key (CMK) with Key Policy & Annual Rotation
# ------------------------------------------------------------------------------
resource "aws_kms_key" "s3_key" {
  description             = "Dedicated KMS Customer Master Key for S3 Data Lake"
  deletion_window_in_days = 30
  enable_key_rotation     = true # Automatically rotates key material every 365 days

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Enable Root Account Administration
      {
        Sid    = "EnableRootAdministration"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      # Allow S3 Service to use Key for cryptographic operations
      {
        Sid    = "AllowS3Encryption"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = { Name = "${var.environment}-s3-kms-key" }
}

resource "aws_kms_alias" "s3_key_alias" {
  name          = "alias/${var.environment}-s3-storage-key"
  target_key_id = aws_kms_key.s3_key.key_id
}

# ------------------------------------------------------------------------------
# 2. Hardened S3 Bucket
# ------------------------------------------------------------------------------
resource "aws_s3_bucket" "secure_bucket" {
  bucket_prefix = "${var.bucket_name_prefix}-${var.environment}-"
  force_destroy = false

  tags = {
    Name        = "${var.bucket_name_prefix}-${var.environment}"
    Environment = var.environment
  }
}

# ------------------------------------------------------------------------------
# 3. Enable Versioning (Ransomware & Accidental Deletion Protection)
# ------------------------------------------------------------------------------
resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.secure_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

# ------------------------------------------------------------------------------
# 4. Enforce S3 Block Public Access (All 4 Guardrails)
# ------------------------------------------------------------------------------
resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket                  = aws_s3_bucket.secure_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ------------------------------------------------------------------------------
# 5. Default Encryption via KMS CMK + Bucket Keys
# ------------------------------------------------------------------------------
resource "aws_s3_bucket_server_side_encryption_configuration" "sse_kms" {
  bucket = aws_s3_bucket.secure_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.s3_key.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true # Reduces KMS request costs by up to 99%!
  }
}

# ------------------------------------------------------------------------------
# 6. Enterprise Bucket Policy (Enforce TLS 1.2+ & KMS Encryption)
# ------------------------------------------------------------------------------
resource "aws_s3_bucket_policy" "strict_policy" {
  bucket     = aws_s3_bucket.secure_bucket.id
  depends_on = [aws_s3_bucket_public_access_block.block_public]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Explicit Deny for all non-HTTPS (unencrypted in transit) requests
      {
        Sid       = "EnforceTLSRequestsOnly"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.secure_bucket.arn,
          "${aws_s3_bucket.secure_bucket.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      # Explicit Deny for uploads not using our dedicated KMS Key
      {
        Sid       = "DenyIncorrectKmsKey"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.secure_bucket.arn}/*"
        Condition = {
          StringNotEqualsIfExists = {
            "s3:x-amz-server-side-encryption-aws-kms-key-id" = aws_kms_key.s3_key.arn
          }
        }
      }
    ]
  })
}

# ------------------------------------------------------------------------------
# 7. Automated S3 Lifecycle Tiering Configuration
# ------------------------------------------------------------------------------
resource "aws_s3_bucket_lifecycle_configuration" "lifecycle_rules" {
  bucket = aws_s3_bucket.secure_bucket.id

  rule {
    id     = "tiering-and-deep-archive-rule"
    status = "Enabled"

    # Transition active current object versions
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    transition {
      days          = 180
      storage_class = "DEEP_ARCHIVE"
    }

    # Manage non-current (older version) objects
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 365 # Delete old superseded versions after 1 year
    }
  }
}
```

### 1.3 `outputs.tf`
```hcl
output "s3_bucket_name" {
  description = "The name of the secured S3 bucket"
  value       = aws_s3_bucket.secure_bucket.bucket
}

output "s3_bucket_arn" {
  description = "The ARN of the secured S3 bucket"
  value       = aws_s3_bucket.secure_bucket.arn
}

output "kms_key_arn" {
  description = "The ARN of the KMS Customer Managed Key"
  value       = aws_kms_key.s3_key.arn
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create the Customer Managed KMS Key (CMK)
1. Open the **AWS KMS Console** (`https://console.aws.amazon.com/kms/`).
2. In the left navigation, click **Customer managed keys** -> **Create key**.
3. **Key type:** Select **Symmetric**. Key usage: **Encrypt and decrypt**. Click **Next**.
4. **Alias:** `production-s3-storage-key`. Click **Next**.
5. **Key administrators:** Select your administrative IAM user/role.
6. **Key users:** Leave default. Click **Next**.
7. Review key policy and click **Finish**.
8. Select the newly created key, go to the **Key rotation** tab, and check **Automatically rotate this KMS key every year**. Click **Save**.

---

### Step 2: Create the Hardened S3 Bucket
1. Open the **Amazon S3 Console** (`https://console.aws.amazon.com/s3/`).
2. Click **Create bucket**.
3. **Bucket name:** `enterprise-secure-lake-prod-94821`.
4. **Bucket Versioning:** Select **Enable**.
5. **Default encryption:**
   * Encryption type: **Server-side encryption with AWS Key Management Service keys (SSE-KMS)**.
   * AWS KMS key: Choose **Choose from your AWS KMS keys**, and pick `alias/production-s3-storage-key`.
   * **Bucket Key:** Select **Enable**.
6. **Block Public Access settings:** Verify **Block all public access** is **CHECKED** (all 4 settings enabled). Click **Create bucket**.

---

### Step 3: Attach Strict Security Bucket Policy
1. In the S3 Console, click on your bucket name.
2. Select the **Permissions** tab. Scroll down to **Bucket policy** and click **Edit**.
3. Paste the strict TLS enforcement policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "EnforceTLSRequestsOnly",
         "Effect": "Deny",
         "Principal": "*",
         "Action": "s3:*",
         "Resource": [
           "arn:aws:s3:::enterprise-secure-lake-prod-94821",
           "arn:aws:s3:::enterprise-secure-lake-prod-94821/*"
         ],
         "Condition": {
           "Bool": {
             "aws:SecureTransport": "false"
           }
         }
       }
     ]
   }
   ```
4. Click **Save changes**.

---

### Step 4: Configure Lifecycle Transition Rule
1. Under the bucket page, select the **Management** tab.
2. Under **Lifecycle rules**, click **Create lifecycle rule**.
3. **Lifecycle rule name:** `tiering-to-glacier-deep-archive`.
4. **Choose a rule scope:** Apply to all objects in the bucket (check acknowledgment).
5. **Lifecycle rule actions:**
   * Check: **Move current versions of objects between storage classes**.
   * Check: **Move noncurrent versions of objects between storage classes**.
6. Under **Transition current versions**:
   * Add transition: Standard-IA after `30` days.
   * Add transition: Glacier Flexible Retrieval after `90` days.
   * Add transition: Glacier Deep Archive after `180` days.
7. Click **Create rule**.
