# AWS Identity & Access Management (IAM), Roles & Least Privilege

IAM controls authentication (AuthN) and authorization (AuthZ) for human identities and machine workloads.

---

## 1. IAM Evaluation Logic & Policy Types

```text
Explicit Deny (Highest Priority) -> Service Control Policy (SCP) -> Permission Boundary -> Identity Policy / Resource Policy -> Default Deny
```

### Policy Categories:
1. **Identity-Based Policies:** Attached to Users, Groups, or Roles.
2. **Resource-Based Policies:** Attached to AWS resources directly (S3 Bucket Policies, KMS Key Policies, SQS Queue Policies).
3. **Permissions Boundaries:** Maximum permissions an identity can be granted (acts as an upper limit guardrail).
4. **Service Control Policies (SCPs):** Organization-level policy guardrails applied across AWS member accounts.

---

## 2. Secure IAM Role & Policy Example (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3EncryptedOperations",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::production-app-vault-2026/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "true"
        }
      }
    },
    {
      "Sid": "EnforceKMSEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/mrk-xxxx-xxxx"
    }
  ]
}
```

* **Best Practice:** Never embed long-term access keys (`AKIA...`) in applications; use IAM Roles with AWS STS temporary credentials (`AssumeRole`) on EC2 / ECS tasks.
