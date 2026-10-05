# AWS CloudTrail Auditing & CloudWatch Observability Guardrails

Auditing and observability are non-negotiable for production cloud environments. When a security breach occurs or a critical resource disappears, you must determine who executed the API call, from what IP address, and at what millisecond. This guide deconstructs **AWS CloudTrail**, **CloudWatch Logs**, **Metric Filters**, and **Composite Alarms**.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Security Camera Footage and the Dashboard Gauges
Imagine managing a high-security bank:
* **AWS CloudTrail (The 24/7 Security CCTV Camera):** Points at every door, safe handle, and computer terminal. Every time an employee touches a door handle or enters a room, CloudTrail records a video clip: *"At 14:02:11, Security Officer John used ID Badge #452 to open the Gold Room door from inside Terminal 3."*
* **AWS CloudWatch (The Airplane Cockpit Instruments):** Monitors real-time vital telemetry. Tells you the fuel level (**Free Memory**), airspeed (**CPU Utilization**), altitude (**Network Throughput**), and sounds a siren (**CloudWatch Alarm**) if the engine temperature exceeds 800 degrees.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### AWS CloudTrail Architecture: Who Did What, Where & When?

CloudTrail records every AWS API call made via the AWS Management Console, AWS SDKs, CLI tools, and internal AWS service calls.

#### 1. Management Events vs. Data Events:
```
+------------------------------------+------------------------------------+
| MANAGEMENT EVENTS (Control Plane)  | DATA EVENTS (Data Plane)           |
+------------------------------------+------------------------------------+
| High-level administrative actions  | Resource-level data operations     |
| Examples:                          | Examples:                          |
| - ec2:RunInstances                 | - s3:GetObject / s3:PutObject      |
| - iam:CreateUser                   | - dynamodb:GetItem / PutItem       |
| - rds:CreateDBInstance             | - lambda:Invoke                    |
+------------------------------------+------------------------------------+
| Enabled by default: YES (90 days)  | Enabled by default: NO!            |
| Free in Event History              | (Must be explicitly turned on)     |
+------------------------------------+------------------------------------+
| Cost: First multi-region copy is   | Cost: $0.10 per 100,000 data       |
| FREE; additional trails cost $2/mo | events (Can generate huge bills!)  |
+------------------------------------+------------------------------------+
```

#### 2. CloudTrail Log File Integrity Validation:
To prevent an attacker from modifying or deleting CloudTrail logs stored in S3 to cover their tracks:
* CloudTrail uses cryptographic hashing (**SHA-256 and RSA signatures**).
* Every hour, CloudTrail writes a **Digest File** containing the cryptographic hashes of every log file delivered in that hour.
* You can run `aws cloudtrail validate-logs` to mathematically prove that no log file was altered or tampered with.

---

### Amazon CloudWatch Architecture

```
[ METRIC PRODUCERS ]
  |-- AWS Hypervisor (EC2 CPU, Network, Disk Read/Write)
  |-- CloudWatch Agent (RAM Utilization, Disk Space, Swap, Log files)
  `-- Custom Metrics (Application Orders per second, Payment Failures)
           |
           v
[ Amazon CloudWatch Telemetry Engine ]
  |
  +---> CloudWatch Logs Groups & Metric Filters (Regex pattern to metric)
  |
  +---> CloudWatch Dashboards (Real-time operational visibility)
  |
  `---> CloudWatch Alarms:
          |-- Static Threshold: CPU > 80% for 3 periods
          |-- Anomaly Detection: Band around expected standard deviation
          `-- Composite Alarm: (AlarmA AND AlarmB) OR AlarmC
```

---

### Transforming Logs into Metrics: CloudWatch Metric Filters

Suppose your application outputs error logs to `/aws/container/payment-api`. You want to trigger an alarm if more than 10 `HTTP 500` errors occur within 1 minute:

1. **Create Metric Filter on Log Group:**
   * Filter Pattern: `[ip, user, timestamp, request, status_code = 5*, size]`
   * Metric Name: `HTTP5xxErrorCount`
   * Metric Value: `1`
2. **Attach CloudWatch Alarm:**
   * Threshold: `HTTP5xxErrorCount >= 10` for 1 evaluation period (60 seconds).
   * Action: Publish to SNS topic `critical-ops-page`.

---

### Composite Alarms (Eliminating Alert Fatigue)

During a major data center event, a single failure can trigger 50 individual alarms (High Latency, High CPU, DB Error, 5xx Spikes), overwhelming the on-call engineer.
A **Composite Alarm** evaluates rules using Boolean logic:

```text
ALARM(AppLatencyHigh) AND ALARM(HTTP5xxRateHigh) AND NOT ALARM(ScheduledMaintenanceWindow)
```

The on-call team receives **exactly one actionable page** instead of 50 noisy notifications!

---

##### 3. Visual Flow Diagram: CloudTrail Security Alerting Pipeline

```
[ Attacker attempts unauthorized API call: iam:CreateUser ]
                             |
                             v
                  [ AWS CloudTrail Engine ]
                             |
                             v (Streams within 5 minutes)
                 [ CloudWatch Logs Group ]
                             |
                             v (Metric Filter matches: eventName = "CreateUser")
               [ CloudWatch Alarm Triggers (State: ALARM) ]
                             |
                             v
               [ Amazon SNS Alert Topic ] ===> PagerDuty / Slack Webhook
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Why doesn't CloudWatch show EC2 Memory (RAM) Utilization by default?"**
  * *The Trap:* EC2 metrics are collected externally by the **AWS Hypervisor (Nitro System)**. The hypervisor can see CPU cycles, network packets, and disk block I/O, but it cannot peek inside the operating system's kernel to see how much RAM is occupied by Java heap!
  * *The Fix:* You must install the **CloudWatch Agent** inside the EC2 operating system, which runs userspace commands and pushes custom memory metrics to CloudWatch.

---

##### 5. Quick Hands-On Check

Query the last 5 administrative IAM modifications executed across your account using CloudTrail via the AWS CLI:

```bash
# Query recent IAM management events from CloudTrail
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=iam.amazonaws.com \
    --max-items 5 \
    --query "Events[*].{EventTime:EventTime,User:Username,Event:EventName}" \
    --output table
```
