# AWS IAM Mastery: Evaluation Logic, Temporary STS Tokens & Conditional Boundaries

AWS Identity and Access Management (IAM) is the central security control plane of everything you build in AWS. There is no concept of "physical perimeter" in the cloud; IAM policies are your security perimeter. This guide breaks down the AWS Policy Evaluation Engine, temporary credentials via STS, conditional context keys, and cross-account trust architectures.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Military Base Security Clearance
Imagine visiting a high-security military intelligence base:
* **The Rule of Explicit Deny:** If the Base Commander issues a decree: *"No one without a hazmat suit may enter Sector 4"* (**Explicit Deny**), it doesn't matter if your badge says "5-Star General" (**AdministratorAccess**); the guard will bar the door. An Explicit Deny always overrules every allow.
* **The Permission Boundary:** Think of this as the maximum physical fence line of the base. Even if you hold a gold key that unlocks every door in the country, if the base gate itself is locked to a specific radius, your key cannot turn locks outside that boundary.
* **AWS STS Temporary Credentials (The Day-Pass RFID Badge):** You do not give developers permanent brass keys to the armory (**Static IAM User Access Keys**). You verify their biometric passport at the front gate (**Single Sign-On / SAML / OIDC**) and hand them an electronic visitor badge (**STS AssumeRole**) that automatically self-destructs and deactivates after 60 minutes.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### The Official AWS Policy Evaluation Decision Tree

When an identity (User, Role, or Service) makes an AWS API request (e.g. `s3:GetObject`), AWS evaluates authorization using this strict sequence:

```
[ INCOMING AWS API REQUEST ]
             |
             v
+-------------------------------------------------------------+
| 1. IS THERE AN EXPLICIT DENY?                               |
|    (In SCP, Boundary, Identity Policy, or Resource Policy)  |
+------------------------------+------------------------------+
                               |
            +------------------+------------------+
            | YES                                 | NO
            v                                     v
     [ EXPLICIT DENY ]           +---------------------------------------+
   ==> REQUEST REJECTED!         | 2. IS THERE AN ORGANIZATIONS SCP?     |
                                 |    (Must be explicitly permitted)     |
                                 +-------------------+-------------------+
                                                     |
                                  +------------------+------------------+
                                  | NO (Blocked)                        | YES (Permitted)
                                  v                                     v
                           [ IMPLICIT DENY ]           +---------------------------------------+
                         ==> REQUEST REJECTED!         | 3. IS THERE A PERMISSION BOUNDARY?    |
                                                       |    (Must be explicitly permitted)     |
                                                       +-------------------+-------------------+
                                                                           |
                                                        +------------------+------------------+
                                                        | NO (Outside)                        | YES (Inside)
                                                        v                                     v
                                                 [ IMPLICIT DENY ]           +---------------------------------------+
                                               ==> REQUEST REJECTED!         | 4. IS THERE AN IDENTITY OR RESOURCE   |
                                                                             |    POLICY ALLOW STATEMENT?            |
                                                                             +-------------------+-------------------+
                                                                                                 |
                                                                              +------------------+------------------+
                                                                              | NO                                  | YES
                                                                              v                                     v
                                                                       [ DEFAULT DENY ]                      [ ACCESS GRANTED! ]
                                                                     ==> REQUEST REJECTED!                  (200 OK Execution)
```

---

### Temporary Credentials: AWS Security Token Service (STS)

Never issue long-term AWS Access Keys (`AKIA...`) to developers or applications. Static keys are leaked onto public GitHub repositories every hour.
Always use **IAM Roles** and **AWS STS**:

#### How `sts:AssumeRole` Works:
1. An identity calls: `aws sts assume-role --role-arn arn:aws:iam::123456789012:role/DeployerRole --role-session-name DeploySession`.
2. AWS STS verifies that the caller matches the Role's **Trust Policy** (`Principal`).
3. STS generates three ephemeral values:
   * **`AccessKeyId`** (starts with `ASIA...`, signaling temporary credentials).
   * **`SecretAccessKey`**
   * **`SessionToken`** (cryptographic token carrying permissions and expiration).
4. Default expiration: **1 hour** (configurable from 15 minutes up to 12 hours).

---

### Deep Dive: Conditional Context Keys

Conditions make policies dynamic and context-aware.

#### 1. Enforcing Multi-Factor Authentication (MFA):
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```

#### 2. Restricting Access to Corporate Public IP Egress:
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "NotIpAddress": {
      "aws:SourceIp": ["198.51.100.0/24", "203.0.113.50/32"]
    }
  }
}
```

#### 3. Restricting S3 Sharing to AWS Organization Members Only:
```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::corporate-shared-bucket/*",
  "Principal": "*",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalOrgID": "o-a1b2c3d4e5"
    }
  }
}
```

---

### Cross-Account IAM Role Architecture

To grant an engineer in Account A (Management: `111111111111`) access to deploy infrastructure in Account B (Production: `222222222222`):

```
ACCOUNT A (Management: 111111111111)                      ACCOUNT B (Production: 222222222222)
+----------------------------------+                      +----------------------------------+
| User: "Alice"                    |                      | IAM Role: "ProductionAdminRole"  |
|                                  |                      |                                  |
| Identity Policy attached:        |                      | 1. Trust Policy (Who can assume):|
| {                                |                      | {                                |
|   "Effect": "Allow",             |                      |   "Effect": "Allow",             |
|   "Action": "sts:AssumeRole",    |==== AssumeRole ====> |   "Principal": {                 |
|   "Resource": "arn:aws:iam::     |                      |     "AWS": "arn:aws:iam::        |
|     222222222222:role/ProdRole"  |                      |             111111111111:root"     |
| }                                |                      |   },                             |
+----------------------------------+                      |   "Action": "sts:AssumeRole"     |
                                                          | }                                |
                                                          |                                  |
                                                          | 2. Permissions Policy:           |
                                                          |    "Action": "ec2:*", "s3:*"     |
                                                          +----------------------------------+
```

---

##### 3. Visual Flow Diagram: Permission Boundary Guardrail

```
[ Developer Identity Policy ]
|-- Granted: s3:*, ec2:*, iam:CreateUser, iam:AttachUserPolicy (Looks like Full Admin!)
         |
         v
[ Permission Boundary Attached to Developer ]
|-- Max Allowed: s3:*, ec2:* (DOES NOT PERMIT iam:*)
         |
         v
[ EFFECTIVE PERMISSION (Intersection of both) ]
|-- Can manipulate S3 and EC2.
`-- CANNOT create IAM users or escalate their own privileges!
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Can an IAM Role be assigned to another IAM Role directly?"**
  * You cannot attach a role to a role. However, an IAM Role can execute `sts:AssumeRole` to assume another IAM Role (Role Chaining). Note: Role chaining limits the maximum session duration to **1 hour**.

---

##### 5. Quick Hands-On Check

Simulate and evaluate an IAM policy without modifying live AWS infrastructure using the IAM Policy Simulator:

```bash
# Simulate whether an IAM role has permission to delete a specific S3 bucket
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:role/DeployerRole \
    --action-names s3:DeleteBucket \
    --resource-arns arn:aws:s3:::production-financial-records \
    --output table
```
