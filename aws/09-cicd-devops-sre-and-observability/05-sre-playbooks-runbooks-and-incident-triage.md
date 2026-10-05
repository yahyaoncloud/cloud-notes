# SRE Reliability Engineering, Incident Response & Runbook Authoring

When production systems fail, chaos ensues if the engineering team lacks structured procedures. Site Reliability Engineering (SRE) treats operations as a software problem. This guide deconstructs **SLIs, SLOs, SLAs, Error Budgets**, the **5-Phase Incident Response Lifecycle**, and how to author deterministic, production-grade **Runbooks**.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Airline Cockpit Emergency Checklist
Imagine a commercial airplane flying at 35,000 feet when an engine warning light flashes red:
* **The SRE Philosophy:** The pilots do not open an engineering textbook and start debating the thermodynamics of jet fuel. They pull out the **Emergency Quick Reference Handbook (Runbook)**.
* **The Runbook:** Step 1: Throttle engine 2 to idle. Step 2: Flip fuel shutoff valve. Step 3: Check fire extinguishing indicator. The steps are deterministic, unambiguous, and can be executed by anyone under extreme adrenaline stress.
* **SLI / SLO / Error Budget (The Fuel Gauge):** The airline promises passengers a safe landing (**SLA**). The flight computer aims for 100% fuel reserve safety (**SLO**). The extra 1,000 gallons in the tank is your **Error Budget**: you can use that extra fuel to fly around a thunderstorm (**Deploy new innovative features**), but if you burn through the safety reserve, the pilot immediately aborts the mission and lands at the nearest runway (**Freeze deployments and fix platform stability**).

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### SLI vs. SLO vs. SLA vs. Error Budgets

```
+----------------------------------------------------------------------------------------------------+
|                                    SRE RELIABILITY METRIC HIERARCHY                                |
+------------------------------------+---------------------------------------------------------------+
| SLI (Service Level Indicator)      | The REAL-TIME MEASUREMENT of system performance.              |
|                                    | Formula: (Successful Good Requests / Total Valid Requests)    |
|                                    | Example: "99.85% of API calls returned in < 200ms in past 1h"|
+------------------------------------+---------------------------------------------------------------+
| SLO (Service Level Objective)      | The INTERNAL TARGET agreed upon between Eng and Product.      |
|                                    | Example: "Our service must maintain 99.9% SLI over 30 days."  |
+------------------------------------+---------------------------------------------------------------+
| SLA (Service Level Agreement)      | The EXTERNAL LEGAL CONTRACT signed with paying customers.     |
|                                    | Stricter consequences: Financial refunds if breached.        |
|                                    | Example: "99.5% uptime; 10% credit if breached."              |
|                                    | (Rule: SLO must ALWAYS be stricter than SLA!).                |
+------------------------------------+---------------------------------------------------------------+
| ERROR BUDGET                       | The ALLOWABLE DOWNTIME during an evaluation window:           |
|                                    | Error Budget = 100% - SLO                                     |
|                                    | For 99.9% SLO: Error budget is 0.1% = ~43 minutes / month.   |
+------------------------------------+---------------------------------------------------------------+
```

#### The Error Budget Policy:
* **If Error Budget > 0:** Feature development continues at maximum velocity.
* **If Error Budget is Exhausted (Hits 0%):** Feature deployments are frozen! 100% of engineering bandwidth pivots to technical debt, reliability refactoring, and automated testing until the rolling budget recovers.

---

### The 5-Phase Incident Response Lifecycle

```
[ 1. DETECTION ] ===> [ 2. TRIAGE & IC ] ===> [ 3. MITIGATION ] ===> [ 4. RESOLUTION ] ===> [ 5. POST-MORTEM ]
```

#### Phase 1: Detection
* Automated alerting via CloudWatch Alarms / PagerDuty. A good alert pages on **symptoms affecting users (High 5xx error rate or high latency)**, not raw server CPU.

#### Phase 2: Triage & Incident Commander (IC)
* Establish an Incident Commander who leads the triage bridge (Slack or Zoom).
* The IC does **not** debug code; the IC coordinates communication, assigns tasks, and shields engineers from executive interruptions.

#### Phase 3: Containment & Mitigation (The Golden Rule)
> **NEVER ATTEMPT TO DEBUG OR FIX CODE DURING AN ACTIVE P1 OUTAGE!**
> **STOP THE BLEEDING FIRST:**
> 1. Roll back the latest deployment immediately.
> 2. Scale out compute capacity (add 50 instances to the ASG).
> 3. Restart crashed services or failover the RDS database.
> 4. Shed non-critical traffic (enable WAF rate limits or degrade secondary features).

#### Phase 4: Resolution & Verification
* Confirm SLIs return to normal baseline for at least 15 minutes before closing the incident bridge.

#### Phase 5: Blameless Post-Mortem & The 5 Whys
* SRE post-mortems assume human error is a symptom of broken systems, not a cause. Focus on guardrails, not blame.
* **The 5 Whys Root Cause Analysis:**
  1. *Why did the database crash?* Max connections were exhausted.
  2. *Why were connections exhausted?* A Lambda function scaled to 2,000 concurrent executions.
  3. *Why did Lambda scale to 2,000?* An external partner sent an unbatched webhook spike.
  4. *Why didn't we buffer the webhooks?* Lambda was directly invoked by API Gateway instead of an SQS queue.
  5. *Action Item:* Place an SQS FIFO queue with an RDS Proxy between API Gateway and the database.

---

### Production Runbook Template

Every critical alert must link directly to an operational Runbook:

```markdown
# Runbook: High ALB 5xx Error Rate Spike (Alert: ALB-5xx-High)

## 1. Summary & Severity
* **Severity:** P1 (High Impact)
* **Description:** Application Load Balancer reports > 5% HTTP 5xx errors over a 5-minute rolling window.
* **Primary Dashboard:** [CloudWatch Web Performance Dashboard](https://console.aws.amazon.com/...)

## 2. Fast Verification Commands
Run from bastion or management terminal:
```bash
# Check if error is 502 (Backend down) or 504 (Backend timeout)
aws elbv2 describe-target-health --target-group-arn <TG-ARN>
# Check backend container error logs in CloudWatch Logs
aws logs filter-log-events --log-group-name /aws/ecs/production --filter-pattern "ERROR" --limit 50
```

## 3. Immediate Mitigation Steps (Choose appropriate scenario):
* **Scenario A: High errors started after a recent deployment (< 30 mins):**
  * Execute instant rollback:
    ```bash
    aws ecs update-service --cluster prod --service api --task-definition api:OLD_REVISION
    ```
* **Scenario B: Database CPU saturated (> 90%):**
  * Force failover to read replica:
    ```bash
    aws rds failover-db-cluster --db-cluster-identifier prod-aurora
    ```
* **Scenario C: Sudden unexpected traffic spike (DDoS or surge):**
  * Increase ASG desired capacity:
    ```bash
    aws autoscaling set-desired-capacity --auto-scaling-group-name prod-asg --desired-capacity 40
    ```

## 4. Escalation Path
* Primary On-Call: @cloud-ops-oncall (PagerDuty)
* Secondary Escalation: Lead Systems Architect (+1-555-0199)
```
