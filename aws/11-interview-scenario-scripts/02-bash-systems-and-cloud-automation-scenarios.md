# Bash Systems & Cloud Automation Interview Scenarios: Practical Scripts, Questions & Answers

Cloud engineers frequently deploy EC2 User Data bootstrap scripts, container entrypoints, and automated server diagnostics. This guide breaks down high-frequency Bash interview scenarios, robust shell programming standards (`set -euo pipefail`), and production execution patterns.

---

## Scenario 1: Production EC2 User Data Bootstrap Script

### The Interview Question
"Write an enterprise-grade EC2 User Data script that configures an Amazon Linux 2023 or RHEL instance into a hardened Web Application node. The script must log all output to a dedicated file, exit gracefully if any command fails, install and configure NGINX, query the AWS IMDSv2 metadata service to extract the instance's public IP and Availability Zone, generate a dynamic index page displaying this info, and send a CloudFormation cfn-signal or health check metric when ready."

### Why Interviewers Ask This
Interviewers use this to identify whether you write amateur shell scripts (which silently fail, get stuck in endless reboots, or leak credentials) versus production scripts that utilize:
1. Strict mode: `set -euo pipefail`.
2. IMDSv2 session tokens (protecting against SSRF vulnerabilities).
3. Log redirection to `/var/log/user-data.log`.
4. Idempotent configuration changes.

### Production Script

```bash
#!/usr/bin/env bash
# ==============================================================================
# Production EC2 Instance Bootstrap Script (Amazon Linux 2023 / RHEL 9)
# ==============================================================================

# Enable strict execution mode:
# -e: Exit immediately if any command exits with a non-zero status.
# -u: Treat unset variables as an error and exit immediately.
# -o pipefail: Return the exit status of the last command in the pipe that failed.
set -euo pipefail

# Redirect all stdout and stderr to both console and a dedicated log file
exec > >(tee -a /var/log/user-data.log | logger -t user-data -s 2>/dev/console) 2>&1

echo "[$(date -u +'%Y-%m-%dT%H:%M:%SZ')] Starting EC2 User-Data Provisioning..."

# Function for clean error handling
trap 'catch_error $? $LINENO' ERR
catch_error() {
    local exit_code=$1
    local line_no=$2
    echo "[ERROR] Command failed at line ${line_no} with exit code ${exit_code}."
    exit "${exit_code}"
}

# 1. Update OS packages and install core runtime utilities
echo "Updating packages and installing dependencies..."
dnf update -y
dnf install -y nginx jq curl

# 2. Securely query AWS Instance Metadata Service Version 2 (IMDSv2)
echo "Querying IMDSv2 for instance metadata..."
# Step A: Request a 6-hour IMDSv2 session token
IMDS_TOKEN=$(curl -s -S -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Step B: Query instance metadata using the token
INSTANCE_ID=$(curl -s -S -H "X-aws-ec2-metadata-token: ${IMDS_TOKEN}" \
    http://169.254.169.254/latest/meta-data/instance-id)
AZ_NAME=$(curl -s -S -H "X-aws-ec2-metadata-token: ${IMDS_TOKEN}" \
    http://169.254.169.254/latest/meta-data/placement/availability-zone)
LOCAL_IPV4=$(curl -s -S -H "X-aws-ec2-metadata-token: ${IMDS_TOKEN}" \
    http://169.254.169.254/latest/meta-data/local-ipv4)

echo "Discovered Instance ID: ${INSTANCE_ID} | AZ: ${AZ_NAME} | Private IP: ${LOCAL_IPV4}"

# 3. Create a dynamic, informative landing page for ALB health checks
cat <<EOF > /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Production Web Application Node</title>
    <style>
        body { font-family: monospace; padding: 2rem; background: #0f172a; color: #f8fafc; }
        .card { background: #1e293b; padding: 1.5rem; border-radius: 8px; border: 1px solid #334155; }
        .field { margin-bottom: 0.5rem; }
        .val { color: #38bdf8; font-weight: bold; }
    </style>
</head>
<body>
    <div class="card">
        <h2>Node Health: Operational</h2>
        <div class="field">Instance ID: <span class="val">${INSTANCE_ID}</span></div>
        <div class="field">Availability Zone: <span class="val">${AZ_NAME}</span></div>
        <div class="field">Private IPv4: <span class="val">${LOCAL_IPV4}</span></div>
        <div class="field">Boot Timestamp: <span class="val">$(date -u +'%Y-%m-%dT%H:%M:%SZ')</span></div>
    </div>
</body>
</html>
EOF

# Ensure appropriate permissions
chown nginx:nginx /usr/share/nginx/html/index.html
chmod 644 /usr/share/nginx/html/index.html

# 4. Enable and start NGINX web server
echo "Starting and enabling NGINX service..."
systemctl enable nginx
systemctl start nginx

# 5. Verify local web server is responding via HTTP 200 OK
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/)
if [ "${HTTP_STATUS}" -ne 200 ]; then
    echo "[ERROR] Local NGINX check returned HTTP ${HTTP_STATUS} instead of 200."
    exit 1
fi

echo "[$(date -u +'%Y-%m-%dT%H:%M:%SZ')] User-Data Bootstrap Completed Successfully."
```

### Interview Explanation & Discussion Points
* **Why IMDSv2 over IMDSv1:** In IMDSv1, any process or SSRF vulnerability could query `http://169.254.169.254/latest/meta-data/iam/security-credentials/` via a plain GET request and steal IAM role credentials. IMDSv2 requires a `PUT` request with custom headers to issue a short-lived token, stopping SSRF attacks cold.
* **The `exec > >(tee ...)` trick:** By redirecting stdout and stderr through `tee` to `/var/log/user-data.log`, every line printed by any command is captured for post-mortem debugging if an EC2 instance fails to launch.

---

## Scenario 2: High-Performance Log File Parser and Anomaly Detection

### The Interview Question
"Given a 10 GB NGINX access log on a live Linux server, write a bash script that parses the log to find:
1. The top 10 IP addresses making requests.
2. The number of 5xx HTTP server errors vs 2xx/3xx/4xx responses.
3. The top 5 endpoints generating 500/502/503/504 errors.
How do you ensure the script does not load the entire file into memory or exhaust disk I/O?"

### Production Script

```bash
#!/usr/bin/env bash
# ==============================================================================
# NGINX Access Log Analyzer: Memory-Safe Streaming Processor
# ==============================================================================
set -euo pipefail

LOG_FILE="${1:-/var/log/nginx/access.log}"

if [ ! -f "${LOG_FILE}" ]; then
    echo "Usage: $0 <path-to-access.log>"
    echo "Error: File '${LOG_FILE}' does not exist."
    exit 1
fi

echo "======================================================================"
echo "Analyzing Access Log: ${LOG_FILE}"
echo "======================================================================"

# Standard NGINX Combined Log Format:
# $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"
# $1: IP, $7: URL Path, $9: HTTP Status Code

echo -e "\n--- Top 10 Requesting Client IP Addresses ---"
# Streaming awk extraction -> sort -> uniq count -> numeric reverse sort -> head
awk '{print $1}' "${LOG_FILE}" | sort | uniq -c | sort -nr | head -n 10

echo -e "\n--- HTTP Status Code Class Breakdown ---"
awk '{
    status = $9;
    if (status ~ /^2/) { class_2xx++ }
    else if (status ~ /^3/) { class_3xx++ }
    else if (status ~ /^4/) { class_4xx++ }
    else if (status ~ /^5/) { class_5xx++ }
    else { other++ }
}
END {
    printf "2xx (Success):       %8d\n", class_2xx;
    printf "3xx (Redirection):   %8d\n", class_3xx;
    printf "4xx (Client Error):  %8d\n", class_4xx;
    printf "5xx (Server Error):  %8d\n", class_5xx;
    printf "Other/Malformed:     %8d\n", other;
}' "${LOG_FILE}"

echo -e "\n--- Top 5 Endpoints Triggering 5xx Server Errors ---"
awk '$9 ~ /^5/ {print $7}' "${LOG_FILE}" | sort | uniq -c | sort -nr | head -n 5

echo -e "\nAnalysis complete."
```

### Interview Explanation & Discussion Points
* **Stream Processing vs In-Memory Buffering:** Highlight that `awk` processes input line-by-line using constant memory regardless of whether the log file is 10 MB or 100 GB.
* **Large File Optimization:** Mention: "If the log is 50 GB+, sorting the entire stream with `sort` will spill over to temporary disk space. To optimize for high-throughput streaming in production, I would use an associative array in `awk` to aggregate frequencies in a single pass without any external pipe sorting."

---

## Scenario 3: Automated Disk Space Alerting and Emergency Inode Reclaim

### The Interview Question
"A root filesystem on an EC2 instance frequently hits 95% capacity because developers let core dumps or old logs accumulate. Write an automated monitoring script to run via cron that checks root mount usage, sends an emergency alert via AWS SNS if disk space exceeds 90% or inodes exceed 85%, and automatically compresses or deletes files in `/var/log` older than 7 days."

### Production Script

```bash
#!/usr/bin/env bash
# ==============================================================================
# Automated Disk Space & Inode Monitor with Emergency Retention Cleanup
# ==============================================================================
set -euo pipefail

DISK_THRESHOLD=90
INODE_THRESHOLD=85
SNS_TOPIC_ARN="arn:aws:sns:us-east-1:123456789012:cloud-ops-alerts"
AWS_REGION="us-east-1"

# 1. Query disk usage percentage on root filesystem (/)
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

# 2. Query inode usage percentage on root filesystem (/)
INODE_USAGE=$(df -i / | awk 'NR==2 {print $5}' | tr -d '%')

echo "[$(date -u +'%Y-%m-%d %H:%M:%S')] Checked /: Disk: ${DISK_USAGE}%, Inodes: ${INODE_USAGE}%"

ALERT_TRIGGERED=false
MESSAGE=""

if [ "${DISK_USAGE}" -ge "${DISK_THRESHOLD}" ]; then
    ALERT_TRIGGERED=true
    MESSAGE="WARNING: Root disk usage is at ${DISK_USAGE}% (Threshold: ${DISK_THRESHOLD}%)."
fi

if [ "${INODE_USAGE}" -ge "${INODE_THRESHOLD}" ]; then
    ALERT_TRIGGERED=true
    MESSAGE="${MESSAGE} WARNING: Root inode usage is at ${INODE_USAGE}% (Threshold: ${INODE_THRESHOLD}%)."
fi

# Emergency Cleanup if alert triggered
if [ "${ALERT_TRIGGERED}" = true ]; then
    echo "[ACTION] Disk/Inode limit breached. Triggering automated cleanup..."

    # Delete archived/compressed logs older than 7 days in /var/log
    find /var/log -type f \( -name "*.gz" -o -name "*.1" -o -name "*.old" \) -mtime +7 -delete

    # Recalculate usage after cleanup
    NEW_DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')
    FINAL_PAYLOAD="${MESSAGE} Automated cleanup purged logs > 7 days. Disk dropped to ${NEW_DISK_USAGE}%."

    # Publish notification via AWS CLI if installed
    if command -v aws >/dev/null 2>&1; then
        aws sns publish \
            --region "${AWS_REGION}" \
            --topic-arn "${SNS_TOPIC_ARN}" \
            --subject "CRITICAL: EC2 Disk Alert on $(hostname)" \
            --message "${FINAL_PAYLOAD}" || true
    fi
fi
```
