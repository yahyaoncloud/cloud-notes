# Enterprise Python Boto3 & Bash Automation Scripting Standards

Writing one-off scripts on a laptop is completely different from engineering production automation that executes inside Lambda functions, CI/CD runners, and EC2 bootstrap sequences. Production scripts must handle API throttling, paginated datasets, transient network failures, and deterministic error trapping.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Fragile Glass Pipe vs. The Industrial Steel Pipeline
* **Amateur Scripting:** Like connecting two water pumps with thin glass tubing. It works on day one when the water pressure is low. But the moment water pressure surges (**AWS API Rate Throttling: `RequestLimitExceeded`**) or an unexpected pebble enters the pipe (**Unset variable or missing JSON key**), the glass shatters completely, the script crashes silently halfway through, and half your servers are left in a broken, half-provisioned state.
* **Production Scripting:** An industrial reinforced steel pipeline. It has automated pressure-relief valves (**Exponential Backoff & Retries**), flow meters at every junction (**Standardized Logging to stdout/file**), emergency shut-off levers (**Bash `trap ERR` and Python `try/except ClientError`**), and a motorized conveyor that processes cargo in controlled batches (**Paginators**).

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Python Boto3 Engineering Standards

#### 1. Client vs. Resource vs. Session:
* **`boto3.Session`:** Stores configuration state (credentials, AWS region, profile). Always create an explicit session when managing multi-region or cross-account operations.
* **`boto3.client`:** The low-level, official 1-to-1 mapping to AWS REST APIs. **Always use client in production code.**
* **`boto3.resource`:** A higher-level object-oriented abstraction. *Warning:* AWS has stopped developing new features for `resource`, and modern services (like Athena, Bedrock, etc.) have zero resource support.

#### 2. Automatic Exponential Backoff & Retries:
By default, Boto3 retries failed requests up to 4 times. For high-volume enterprise automation, configure adaptive client retries to handle AWS API rate limits:

```python
import boto3
from botocore.config import Config

# Configure client with adaptive retry mode and exponential backoff
custom_config = Config(
    region_name='us-east-1',
    retries={
        'max_attempts': 10,
        'mode': 'adaptive'  # Automatically throttles request rate based on 429 responses
    },
    connect_timeout=5,
    read_timeout=10
)

ec2_client = boto3.client('ec2', config=custom_config)
```

#### 3. Paginators (Never Rely on Single Responses):
AWS APIs truncate results at 1,000 items (or 100 items for some services). A script that does `ec2.describe_instances()` will silently drop all instances beyond the first 1,000!

```python
# The Production Pattern for Large Scale Accounts
paginator = ec2_client.get_paginator('describe_instances')
page_iterator = paginator.paginate(
    Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
)

for page in page_iterator:
    for reservation in page['Reservations']:
        for instance in reservation['Instances']:
            print(f"Discovered Instance: {instance['InstanceId']}")
```

#### 4. Waiters (Eliminating `time.sleep` Anti-Patterns):
Never write `time.sleep(60)` waiting for an EC2 instance or RDS snapshot to be ready! Use **Boto3 Waiters**:

```python
# Launch instance
res = ec2_client.run_instances(ImageId='ami-0123456789', MinCount=1, MaxCount=1)
instance_id = res['Instances'][0]['InstanceId']

print(f"Waiting for {instance_id} to reach 'running' and pass 2/2 status checks...")
waiter = ec2_client.get_waiter('instance_status_ok')
# Polls every 15 seconds until status checks pass or 40 attempts exceed
waiter.wait(InstanceIds=[instance_id])
print(f"Instance {instance_id} is fully online and ready!")
```

#### 5. Defensive Error Handling:
Always catch `botocore.exceptions.ClientError` and parse the specific AWS Error Code:

```python
from botocore.exceptions import ClientError

try:
    s3_client.delete_bucket(Bucket='protected-company-bucket')
except ClientError as err:
    error_code = err.response['Error']['Code']
    if error_code == 'BucketNotEmpty':
        print("ERROR: Cannot delete bucket. Must empty objects first.")
    elif error_code == 'NoSuchBucket':
        print("Bucket already deleted.")
    else:
        print(f"Unexpected AWS API failure: {err.response['Error']['Message']}")
```

---

### Bash Systems Scripting Standards

Every production Bash script in AWS (such as an EC2 User Data script or Docker entrypoint) must adhere to these standards:

#### 1. The Strict Mode Header:
```bash
#!/usr/bin/env bash
# Fail if any command fails (-e), if any variable is unset (-u),
# and prevent errors in pipeline from being masked (-o pipefail)
set -euo pipefail
```

#### 2. The Deterministic Error Trap:
Catch failures with line numbers and exit codes automatically:

```bash
trap 'handle_exit $? $LINENO' EXIT

handle_exit() {
    local exit_code=$1
    local line_number=$2
    if [ "${exit_code}" -ne 0 ]; then
        echo "[FATAL ERROR] Script crashed at line ${line_number} with code ${exit_code}." >&2
    else
        echo "[SUCCESS] Script finished execution cleanly."
    fi
}
```

#### 3. Safe Argument Parsing with `getopts`:
```bash
ENVIRONMENT="dev"
DRY_RUN=false

while getopts "e:d" opt; do
    case "${opt}" in
        e) ENVIRONMENT="${OPTARG}" ;;
        d) DRY_RUN=true ;;
        *) echo "Usage: $0 [-e <environment>] [-d]" >&2; exit 1 ;;
    esac
done

echo "Running deployment for: ${ENVIRONMENT} (DryRun=${DRY_RUN})"
```

#### 4. High-Performance Shell JSON Parsing with `jq`:
Extract values safely without fragile regex or `grep`:

```bash
# Extract private IP address of instance from AWS CLI JSON output
PRIVATE_IP=$(aws ec2 describe-instances --instance-ids i-0123456789 | \
    jq -r '.Reservations[0].Instances[0].PrivateIpAddress // empty')

if [ -z "${PRIVATE_IP}" ]; then
    echo "Error: Private IP not found." >&2
    exit 1
fi
```
