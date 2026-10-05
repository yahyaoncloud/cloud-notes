# AWS KMS Envelope Encryption, Key Management & Secrets Architecture

Data at rest must be cryptographically protected. However, sending gigabytes of database backups or application files directly across AWS network APIs to be encrypted by a hardware security module (HSM) is physically impossible due to API payload limits. This guide deconstructs **AWS Key Management Service (KMS)**, **Envelope Encryption**, and the trade-offs between **AWS Secrets Manager** and **SSM Parameter Store**.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Bank Safety Deposit Box Key and the Master Safe
Imagine you need to lock up a 10-ton gold vault door:
* **The KMS Master Key (The Royal Bank Master Safe):** You have a solid titanium master safe inside a secure, fortified bank vault (**AWS Hardware Security Module / HSM**). This master safe is physically bolted to the concrete floor; it can **never** leave the vault.
* **The 4 KB Constraint:** If you walk up to the bank teller and try to push an entire moving truck through the bank teller's 2-inch glass window, it won't fit (**KMS has a 4,096-byte hard payload limit**).
* **Envelope Encryption (The Small Padlock and Key):**
  1. You ask the bank teller for a disposable brass padlock and key (**Data Encryption Key / DEK**).
  2. The teller generates the padlock and hands you two things: the open physical key (**Plaintext DEK**) and an identical copy of that key locked inside a titanium puzzle box sealed with the Royal Bank Stamp (**Ciphertext DEK**).
  3. You run back to your moving truck, lock the cargo using the brass key, and immediately throw the brass key into an industrial blast furnace (**Shred Plaintext DEK from RAM**)!
  4. You tape the sealed titanium puzzle box to the back of your truck.
  5. Anyone who steals your truck sees only gibberish. To unlock the truck, only the Royal Bank teller can open the puzzle box and regenerate the brass key.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### AWS KMS Key Categories

```
+--------------------------+--------------------------+--------------------------+
| AWS OWNED KEYS           | AWS MANAGED KEYS         | CUSTOMER MANAGED KEYS    |
+--------------------------+--------------------------+--------------------------+
| Used internally by AWS   | Created automatically    | Created and managed by   |
| across multiple accounts | per-service (aws/s3,     | YOUR engineering team    |
| (e.g. S3 internal infra) | aws/rds, aws/ebs)        |                          |
+--------------------------+--------------------------+--------------------------+
| Visible in console: NO   | Visible in console: YES  | Visible in console: YES  |
| Key Policy control: NO   | Key Policy control: NO   | Key Policy control: FULL |
| Automated rotation: YES  | Automated rotation: YES  | Automated rotation: Opt  |
| Cost: FREE               | Cost: FREE               | Cost: $1.00 / month / key|
| Cross-account sharing: NO| Cross-account sharing: NO| Cross-account: YES!      |
+--------------------------+--------------------------+--------------------------+
```

---

### Envelope Encryption Step-by-Step

Because KMS APIs can only encrypt strings up to **4 KB (4,096 bytes)**, AWS services (S3, EBS, RDS) use **Envelope Encryption** for all data storage:

```
============================== ENCRYPTION WORKFLOW ==============================
1. Application calls: kms:GenerateDataKey(KeyId="arn:aws:kms:.../my-key")
2. AWS KMS generates a 256-bit symmetric AES key and returns TWO copies:
   - Copy A: Plaintext Data Key (e.g. "x8F2...91A")
   - Copy B: Ciphertext Encrypted Data Key (Encrypted under the KMS Root Key)
3. Application uses Plaintext Data Key to encrypt the 50 GB database backup in RAM.
4. Application IMMEDIATELY zeroes out / deletes the Plaintext Data Key from RAM!
5. Application saves the encrypted file + prepends the Ciphertext Encrypted Data Key to disk.

============================== DECRYPTION WORKFLOW ==============================
1. Application reads the Ciphertext Encrypted Data Key from the file header.
2. Application calls: kms:Decrypt(CiphertextBlob=EncryptedDataKey)
3. AWS KMS decrypts the key inside its secure HSM and returns the Plaintext Data Key.
4. Application uses Plaintext Data Key to decrypt the 50 GB file.
5. Application purges Plaintext Data Key from RAM.
```

---

### KMS Key Policies (The Ultimate Decider)

Unlike other AWS resources where an IAM identity policy is sufficient:
> **If an IAM policy grants `kms:*`, but the KMS Key Policy does not explicitly permit the account or principal, access is DENIED!**

A KMS Key Policy is the primary authorization mechanism for a Customer Managed Key:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableRootPermissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowAppDecryption",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ProductionAppRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### AWS Secrets Manager vs. AWS Systems Manager Parameter Store

```
+------------------------------------+------------------------------------+
| AWS SECRETS MANAGER                | SSM PARAMETER STORE                |
+------------------------------------+------------------------------------+
| Cost: $0.40 per secret / month     | Cost: Standard tier is 100% FREE!  |
| + $0.05 per 10,000 API calls       | ($0.05/mo for Advanced Tier)       |
+------------------------------------+------------------------------------+
| Automatic Rotation: NATIVE         | Automatic Rotation: NO             |
| Out-of-the-box integration with RDS| (Requires custom manual Lambda)    |
| to rotate DB passwords seamlessly  |                                    |
+------------------------------------+------------------------------------+
| Cross-Account Sharing: YES         | Cross-Account Sharing: Requires RAM|
| (Direct resource policy on secret) | (Resource Access Manager)          |
+------------------------------------+------------------------------------+
| Max Payload: 64 KB                 | Max Payload: 4 KB (Standard)       |
|                                    | 8 KB (Advanced)                    |
+------------------------------------+------------------------------------+
| Best for: Database passwords,      | Best for: Environment configs,     |
| OAuth tokens, dynamic rotation     | feature flags, static license keys |
+------------------------------------+------------------------------------+
```

---

##### 3. Visual Flow Diagram: Automated Database Password Rotation

```
[ Scheduled Timer (e.g. 30 Days) ]
                |
                v
      [ AWS Secrets Manager ]
                |
                v (Invokes)
    [ AWS Rotation Lambda Function ]
      |
      |-- 1. Creates new random password
      |-- 2. Connects to RDS DB and alters user password
      |-- 3. Tests new credentials against database
      `-- 4. Commits new password version in Secrets Manager!
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"I locked myself out of my own KMS Key!"**
  * If you edit a KMS Key Policy and forget to include `"Principal": { "AWS": "arn:aws:iam::123456789012:root" }`, **no one in the AWS account (including the root user and administrators) can modify or delete the key!**
  * You must submit an emergency ticket to AWS Support to review and reset the key policy!

---

##### 5. Quick Hands-On Check

Generate a fresh Data Encryption Key (DEK) via the AWS KMS CLI to inspect the returned plaintext and ciphertext blobs:

```bash
# Request a 256-bit AES Data Key from KMS
aws kms generate-data-key \
    --key-id alias/production-data-key \
    --key-spec AES_256 \
    --region us-east-1
```
