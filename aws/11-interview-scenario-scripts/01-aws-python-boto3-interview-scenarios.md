# AWS Python Boto3 Interview Scenarios: Practical Scripts, Questions & Answers

This guide breaks down real-world Python Boto3 scripting challenges asked during Cloud and DevOps Engineer technical interviews. Each scenario includes the technical requirement, why interviewers ask it, the production-grade script with error handling and pagination, and an interview answer walkthrough.

---

## Scenario 1: Automated Audit and Cleanup of Unattached EBS Volumes

### The Interview Question
"Over time, EC2 instances get terminated, but engineers forget to check 'Delete on Termination' for attached EBS volumes. Write an automated Python Boto3 script to scan across all regions, identify all unattached ('available') EBS volumes older than 14 days, tag them with an expiration date, and safely delete volumes that are untagged or older than the retention threshold. How do you handle pagination, region loops, and API rate limits?"

### Why Interviewers Ask This
Interviewers use this to test three things:
1. Understanding of AWS billing leakage (unattached gp3/io2 volumes cost money even when idle).
2. Proper usage of Boto3 `describe_volumes` filters and client vs resource abstractions.
3. Pagination (`NextToken`), cross-region discovery, and defensive error handling (`botocore.exceptions.ClientError`).

### Production Script

```python
#!/usr/bin/env python3
"""
Audit and clean up unattached EBS volumes older than a retention threshold.
Demonstrates: Cross-region iteration, paginators, client error handling, and safe deletion.
"""

import sys
from datetime import datetime, timezone, timedelta
import boto3
from botocore.exceptions import ClientError

RETENTION_DAYS = 14

def get_all_regions():
    """Retrieve all available AWS regions for EC2."""
    ec2_client = boto3.client('ec2', region_name='us-east-1')
    try:
        response = ec2_client.describe_regions(
            AllRegions=False,
            Filters=[{'Name': 'opt-in-status', 'Values': ['opt-in-not-required', 'opted-in']}]
        )
        return [region['RegionName'] for region in response['Regions']]
    except ClientError as err:
        print(f"ERROR: Unable to list regions: {err.response['Error']['Message']}", file=sys.stderr)
        return ['us-east-1']

def audit_unattached_volumes(dry_run=True):
    """Scan and purge unattached EBS volumes across all regions."""
    regions = get_all_regions()
    cutoff_date = datetime.now(timezone.utc) - timedelta(days=RETENTION_DAYS)
    total_reclaimed_gb = 0

    print(f"Starting unattached EBS volume audit. Retention threshold: {RETENTION_DAYS} days. DryRun={dry_run}")

    for region in regions:
        print(f"--- Scanning Region: {region} ---")
        ec2 = boto3.client('ec2', region_name=region)
        paginator = ec2.get_paginator('describe_volumes')

        try:
            # Filter server-side to minimize payload and API overhead
            page_iterator = paginator.paginate(
                Filters=[{'Name': 'status', 'Values': ['available']}]
            )

            for page in page_iterator:
                for volume in page.get('Volumes', []):
                    vol_id = volume['VolumeId']
                    vol_size = volume['Size']
                    create_time = volume['CreateTime']

                    # Check age
                    if create_time < cutoff_date:
                        print(f"Target Volume Found: {vol_id} | Size: {vol_size} GB | Created: {create_time.isoformat()}")
                        total_reclaimed_gb += vol_size

                        if not dry_run:
                            try:
                                ec2.delete_volume(VolumeId=vol_id)
                                print(f"Successfully deleted unattached volume: {vol_id}")
                            except ClientError as del_err:
                                error_code = del_err.response['Error']['Code']
                                if error_code == 'VolumeInUse':
                                    print(f"Skipping {vol_id}: Volume became in-use.")
                                else:
                                    print(f"Failed to delete {vol_id}: {del_err.response['Error']['Message']}")
                        else:
                            print(f"[DRY-RUN] Would delete {vol_id} ({vol_size} GB).")

        except ClientError as err:
            print(f"Error accessing region {region}: {err.response['Error']['Message']}", file=sys.stderr)

    print(f"\nAudit complete. Total storage identified for reclamation: {total_reclaimed_gb} GB.")

if __name__ == '__main__':
    # Default to Dry-Run for safety
    is_dry_run = True
    if len(sys.argv) > 1 and sys.argv[1].lower() == '--execute':
        is_dry_run = False
    audit_unattached_volumes(dry_run=is_dry_run)
```

### Interview Explanation & Discussion Points
* **Client vs. Resource:** Prefer `boto3.client` over `boto3.resource` because the resource abstraction is deprecated or unsupported in newer AWS SDK patterns, whereas the client maps directly to official AWS REST APIs.
* **Server-Side Filtering:** Instead of downloading thousands of volumes and filtering in Python, use `Filters=[{'Name': 'status', 'Values': ['available']}]`. This reduces network payload and API throttling risk.
* **Paginators:** Always use `ec2.get_paginator('describe_volumes')` instead of relying on a single response dict. In production accounts with thousands of volumes, single calls drop results after 1,000 items.

---

## Scenario 2: S3 Bucket Security Enforcer (Block Public Access & Bucket Policies)

### The Interview Question
"Write a script that audits all S3 buckets in an organization's account, checks whether Public Access Block is enabled, verifies if default SSE encryption is configured, and remediates compliant buckets by enabling Public Access Block automatically."

### Why Interviewers Ask This
Public S3 buckets remain the number one source of cloud data breaches. Interviewers want to see if you can programmatically inspect AWS resource policies, handle missing configurations without crashing, and apply least-privilege remediation.

### Production Script

```python
#!/usr/bin/env python3
"""
S3 Bucket Security Compliance Checker & Remediator.
Audits: S3 Block Public Access, Default SSE Encryption.
Remediates: Enforces S3 Public Access Block if missing.
"""

import sys
import boto3
from botocore.exceptions import ClientError

s3_client = boto3.client('s3')

def check_bucket_encryption(bucket_name):
    """Check if default server-side encryption is active on the bucket."""
    try:
        enc_res = s3_client.get_bucket_encryption(Bucket=bucket_name)
        rules = enc_res.get('ServerSideEncryptionConfiguration', {}).get('Rules', [])
        if rules:
            sse_algorithm = rules[0]['ApplyServerSideEncryptionByDefault']['SSEAlgorithm']
            return True, sse_algorithm
    except ClientError as err:
        if err.response['Error']['Code'] == 'ServerSideEncryptionConfigurationNotFoundError':
            return False, "None"
        return False, f"Error: {err.response['Error']['Code']}"
    return False, "Unknown"

def check_and_enforce_public_access_block(bucket_name, auto_remediate=False):
    """Audit S3 Public Access Block; optionally apply strict blocking."""
    is_protected = False
    try:
        pab_res = s3_client.get_public_access_block(Bucket=bucket_name)
        config = pab_res['PublicAccessBlockConfiguration']
        if (config['BlockPublicAcls'] and 
            config['IgnorePublicAcls'] and 
            config['BlockPublicPolicy'] and 
            config['RestrictPublicBuckets']):
            is_protected = True
    except ClientError as err:
        if err.response['Error']['Code'] == 'NoSuchPublicAccessBlockConfiguration':
            is_protected = False
        else:
            return False, f"API Error: {err.response['Error']['Code']}"

    if not is_protected and auto_remediate:
        try:
            s3_client.put_public_access_block(
                Bucket=bucket_name,
                PublicAccessBlockConfiguration={
                    'BlockPublicAcls': True,
                    'IgnorePublicAcls': True,
                    'BlockPublicPolicy': True,
                    'RestrictPublicBuckets': True
                }
            )
            return True, "Remediated (Enforced)"
        except ClientError as rem_err:
            return False, f"Remediation Failed: {rem_err.response['Error']['Message']}"

    return is_protected, ("Compliant" if is_protected else "NON-COMPLIANT: Missing Public Access Block")

def run_s3_compliance_audit(auto_remediate=False):
    """Enumerate all buckets and output compliance matrix."""
    try:
        buckets = s3_client.list_buckets().get('Buckets', [])
    except ClientError as err:
        print(f"FATAL: Unable to list buckets: {err.response['Error']['Message']}", file=sys.stderr)
        sys.exit(1)

    print(f"{'BUCKET NAME':<40} | {'ENCRYPTION':<15} | {'PUBLIC ACCESS BLOCK':<35}")
    print("-" * 95)

    for bucket in buckets:
        name = bucket['Name']
        has_enc, enc_type = check_bucket_encryption(name)
        has_pab, pab_status = check_and_enforce_public_access_block(name, auto_remediate)
        print(f"{name:<40} | {enc_type:<15} | {pab_status:<35}")

if __name__ == '__main__':
    remediate = '--remediate' in sys.argv
    run_s3_compliance_audit(auto_remediate=remediate)
```

### Interview Explanation & Discussion Points
* **Handling 404 Equivalents in Boto3:** In AWS S3 APIs, if a configuration does not exist (like encryption or public access block), the API does not return `None` or `False`. It raises a `ClientError` with error codes like `ServerSideEncryptionConfigurationNotFoundError` or `NoSuchPublicAccessBlockConfiguration`. An interview candidate must catch these specifically rather than doing a bare `except Exception:`.
* **Remediation Guardrails:** Mention in the interview: "In production, automated remediation is triggered via AWS Config Rule Evaluation and an AWS Systems Manager (SSM) Automation Document, or via an EventBridge rule calling an audited Lambda function with SNS alerting."

---

## Scenario 3: Automated AMI Backup & Snapshot Rotation with Retention Tracking

### The Interview Question
"Developers need daily AMI backups of stateful EC2 instances tagged `Backup=Daily`. Write an automated script that creates an AMI of these instances without rebooting, tags the AMI with an expiration timestamp, and searches for past AMIs created by this script whose expiration date has passed to deregister the AMI and purge its underlying EBS snapshots."

### Production Script

```python
#!/usr/bin/env python3
"""
Automated EC2 AMI Snapshot Lifecycle Manager.
Creates: Tagged AMIs for instances with 'Backup=Daily'.
Cleans up: Expired AMIs and orphan snapshots.
"""

from datetime import datetime, timezone, timedelta
import boto3
from botocore.exceptions import ClientError

RETENTION_DAYS = 7
TAG_KEY = 'Backup'
TAG_VAL = 'Daily'

ec2 = boto3.client('ec2')

def create_daily_backups():
    """Locate tagged EC2 instances and create AMIs."""
    now = datetime.now(timezone.utc)
    expiry = now + timedelta(days=RETENTION_DAYS)
    expiry_str = expiry.strftime('%Y-%m-%d')

    filters = [
        {'Name': f'tag:{TAG_KEY}', 'Values': [TAG_VAL]},
        {'Name': 'instance-state-name', 'Values': ['running', 'stopped']}
    ]

    try:
        instances = ec2.describe_instances(Filters=filters)
    except ClientError as err:
        print(f"Error describing instances: {err}")
        return

    for reservation in instances.get('Reservations', []):
        for inst in reservation.get('Instances', []):
            inst_id = inst['InstanceId']
            # Extract Name tag if present
            name_tag = next((t['Value'] for t in inst.get('Tags', []) if t['Key'] == 'Name'), inst_id)
            ami_name = f"backup-{name_tag}-{now.strftime('%Y%m%d-%H%M%S')}"

            try:
                print(f"Creating AMI for instance {inst_id} ({name_tag})...")
                ami_res = ec2.create_image(
                    InstanceId=inst_id,
                    Name=ami_name,
                    Description=f"Automated daily backup for {inst_id}",
                    NoReboot=True,  # Crucial for zero application downtime
                    TagSpecifications=[
                        {
                            'ResourceType': 'image',
                            'Tags': [
                                {'Key': 'CreatedBy', 'Value': 'AutomatedBackupScript'},
                                {'Key': 'ExpirationDate', 'Value': expiry_str},
                                {'Key': 'SourceInstance', 'Value': inst_id}
                            ]
                        }
                    ]
                )
                print(f"Triggered AMI creation: {ami_res['ImageId']}")
            except ClientError as err:
                print(f"Failed to create AMI for {inst_id}: {err.response['Error']['Message']}")

def purge_expired_backups():
    """Find expired AMIs created by this script, deregister them, and delete associated snapshots."""
    today_str = datetime.now(timezone.utc).strftime('%Y-%m-%d')

    filters = [
        {'Name': 'tag:CreatedBy', 'Values': ['AutomatedBackupScript']},
        {'Name': 'state', 'Values': ['available']}
    ]

    try:
        images = ec2.describe_images(Owners=['self'], Filters=filters).get('Images', [])
    except ClientError as err:
        print(f"Error querying AMIs: {err}")
        return

    for img in images:
        image_id = img['ImageId']
        # Extract ExpirationDate tag
        exp_date_tag = next((t['Value'] for t in img.get('Tags', []) if t['Key'] == 'ExpirationDate'), None)

        if exp_date_tag and exp_date_tag <= today_str:
            print(f"AMI {image_id} has expired (Expiration: {exp_date_tag}). Beginning teardown.")

            # Record snapshot IDs before deregistering the AMI
            snapshot_ids = [
                mapping['Ebs']['SnapshotId']
                for mapping in img.get('BlockDeviceMappings', [])
                if 'Ebs' in mapping and 'SnapshotId' in mapping['Ebs']
            ]

            # 1. Deregister the AMI
            try:
                ec2.deregister_image(ImageId=image_id)
                print(f"Deregistered AMI {image_id}")
            except ClientError as err:
                print(f"Could not deregister {image_id}: {err.response['Error']['Message']}")
                continue

            # 2. Delete the associated snapshots
            for snap_id in snapshot_ids:
                try:
                    ec2.delete_snapshot(SnapshotId=snap_id)
                    print(f"Deleted underlying snapshot {snap_id}")
                except ClientError as snap_err:
                    print(f"Failed to delete snapshot {snap_id}: {snap_err.response['Error']['Message']}")

if __name__ == '__main__':
    create_daily_backups()
    purge_expired_backups()
```

### Interview Explanation & Discussion Points
* **The Snapshot Orphan Bug:** A classic interview trap: "When you run `ec2.deregister_image()`, does AWS delete the EBS snapshots automatically?" The answer is **NO**. AWS leaves the snapshots intact. If you don't parse `BlockDeviceMappings` to collect the snapshot IDs *before* or *during* deregistration, your AWS account will accumulate hundreds of orphaned snapshots costing significant monthly storage fees.

---

## Scenario 4: Dynamic Security Group IP Whitelist Synchronizer

### The Interview Question
"Our office internet has a dynamic public IP or an external monitoring vendor periodically updates their webhook IP list from a JSON URL. Write a script to fetch the current IP list, inspect an existing Security Group, add any newly added IPs with appropriate descriptions, and remove obsolete IPs that are no longer in the vendor's list."

### Production Script

```python
#!/usr/bin/env python3
"""
Dynamic Security Group Ingress Rule Synchronizer.
Reconciles Security Group rules against a target set of authorized CIDR blocks.
"""

import sys
import boto3
from botocore.exceptions import ClientError

TARGET_PORT = 443
PROTOCOL = 'tcp'
SG_ID = 'sg-0123456789abcdef0'

# Simulated dynamic IP source (could be fetched from requests.get(url).json())
ALLOWED_IPS = {'203.0.113.10/32', '198.51.100.24/32', '192.0.2.50/32'}

ec2 = boto3.client('ec2')

def reconcile_security_group_rules(sg_id, target_port, protocol, desired_cidrs):
    """Ensure that only desired CIDRs have ingress access on the specified port."""
    try:
        response = ec2.describe_security_groups(GroupIds=[sg_id])
        sg = response['SecurityGroups'][0]
    except ClientError as err:
        print(f"Error fetching Security Group {sg_id}: {err}")
        return

    # Extract existing CIDRs for the given port and protocol
    existing_cidrs = set()
    for rule in sg.get('IpPermissions', []):
        if rule.get('FromPort') == target_port and rule.get('IpProtocol') == protocol:
            for ip_range in rule.get('IpRanges', []):
                existing_cidrs.add(ip_range['CidrIp'])

    print(f"Existing CIDRs on {protocol}:{target_port}: {existing_cidrs}")
    print(f"Desired CIDRs on {protocol}:{target_port}:  {desired_cidrs}")

    # Determine diffs
    cidrs_to_add = desired_cidrs - existing_cidrs
    cidrs_to_revoke = existing_cidrs - desired_cidrs

    # Authorize new rules
    if cidrs_to_add:
        ip_permissions_to_add = [
            {
                'IpProtocol': protocol,
                'FromPort': target_port,
                'ToPort': target_port,
                'IpRanges': [{'CidrIp': cidr, 'Description': 'Dynamic Automated Ingress'} for cidr in cidrs_to_add]
            }
        ]
        try:
            ec2.authorize_security_group_ingress(GroupId=sg_id, IpPermissions=ip_permissions_to_add)
            print(f"Authorized new CIDRs: {cidrs_to_add}")
        except ClientError as err:
            print(f"Failed to authorize rules: {err.response['Error']['Message']}")

    # Revoke obsolete rules
    if cidrs_to_revoke:
        ip_permissions_to_revoke = [
            {
                'IpProtocol': protocol,
                'FromPort': target_port,
                'ToPort': target_port,
                'IpRanges': [{'CidrIp': cidr} for cidr in cidrs_to_revoke]
            }
        ]
        try:
            ec2.revoke_security_group_ingress(GroupId=sg_id, IpPermissions=ip_permissions_to_revoke)
            print(f"Revoked obsolete CIDRs: {cidrs_to_revoke}")
        except ClientError as err:
            print(f"Failed to revoke rules: {err.response['Error']['Message']}")

    if not cidrs_to_add and not cidrs_to_revoke:
        print("Security group is already in desired state. No changes applied.")

if __name__ == '__main__':
    reconcile_security_group_rules(SG_ID, TARGET_PORT, PROTOCOL, ALLOWED_IPS)
```

### Key Takeaway for Interviews
State reconciliation using mathematical set operations (`desired - existing` and `existing - desired`) demonstrates **idempotency**. An interview answer emphasizing idempotency tells the hiring manager you write safe automation that can run every 5 minutes in a cron or Lambda without corrupting infrastructure state.
