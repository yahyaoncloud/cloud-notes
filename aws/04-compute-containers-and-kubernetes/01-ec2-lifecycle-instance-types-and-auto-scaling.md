# Amazon EC2 Lifecycle, Instance Topologies & Auto Scaling Groups (ASG)

Amazon Elastic Compute Cloud (EC2) provides resizable compute capacity in the cloud. As a cloud engineer, choosing the right instance family, balancing On-Demand vs. Spot economics, and engineering self-healing, horizontally scalable Auto Scaling Groups (ASG) directly impacts application latency and company cloud spend.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: Renting Fleet Vehicles for a Delivery Company
Imagine managing a delivery company:
* **Instance Families:**
  * For standard courier packages, you rent standard family sedans (**General Purpose: `m` series**).
  * For express speed deliveries that need maximum engine horsepower, you rent lightweight sports cars (**Compute Optimized: `c` series**).
  * For hauling heavy, massive liquid tanks, you rent flatbed tractor-trailers with enormous carrying capacity (**Memory Optimized: `r` series**).
* **Purchasing Models:**
  * **On-Demand:** Walking into a car rental agency and paying standard daily rates. You can return it whenever you want, but you pay maximum price.
  * **Savings Plans / Reserved Instances:** Signing a 1-year or 3-year contract promising to keep 10 vans running every day in exchange for a 60% discount.
  * **Spot Instances:** Renting standby rental cars that are sitting empty in the parking lot for an 80% discount—with the explicit agreement that if a regular paying customer walks in, the agency can tow your car away with only 2 minutes notice!
* **Auto Scaling Group (ASG):** An automated dispatcher that counts the customer order backlog. If orders spike, it automatically calls the rental agency to deploy 5 more vans; when the rush subsides, it returns them to stop the bill.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### EC2 Instance Family Naming Decoded

Take `m6i.2xlarge` or `c7g.xlarge`:
* **First Letter (`m` / `c` / `r`):** The workload archetype:
  * **`m` (General Purpose):** Balanced 4:1 ratio of RAM to vCPU (e.g. 8 vCPU, 32 GB RAM). Great for web apps.
  * **`c` (Compute Optimized):** High clock speed, 2:1 ratio of RAM to vCPU (e.g. 8 vCPU, 16 GB RAM). Ideal for video encoding, batch processing, gaming servers.
  * **`r` (Memory Optimized):** Massive RAM, 8:1 ratio of RAM to vCPU (e.g. 8 vCPU, 64 GB RAM). High-performance databases, in-memory caches (Redis).
  * **`i` (Storage Optimized):** Direct NVMe SSD attachments for high random IOPS (NoSQL databases, Cassandra).
  * **`g` / `p` (Accelerated / GPU):** NVIDIA Tensor Core GPUs for AI/ML training and inference.
* **Generation Number (`6` / `7`):** The hardware generation (higher is newer, faster, and cheaper).
* **Processor Architecture Letter (`i` / `a` / `g`):**
  * `i`: Intel Xeon Scalable
  * `a`: AMD EPYC
  * `g`: **AWS Graviton (Custom ARM64 architecture)** - up to 40% better price/performance.
* **Size (`xlarge`, `2xlarge`, `metal`):** The allocation of vCPU cores, memory, and network throughput.

---

### Purchasing Models & Financial Engineering

```
+------------------+------------------+------------------+------------------+
| ON-DEMAND        | SAVINGS PLANS    | SPOT INSTANCES   | DEDICATED HOSTS  |
+------------------+------------------+------------------+------------------+
| Maximum cost     | 40% - 72% off    | Up to 90% off    | Maximum cost     |
| Pay per second   | 1 or 3 yr commit | Unused capacity  | Physical server  |
| No commitment    | Hourly $/hr spend| Can be terminated| Dedicated to you |
| Ideal for: Dev,  | Ideal for: Core  | with 2-min notice| Compliance / BYOL|
| erratic spiky app| steady-state prod| Batch, stateless | strict licensing |
+------------------+------------------+------------------+------------------+
```

#### Production Spot Fleet Strategy:
Never run a single-instance stateful database on Spot!
* **Where to use Spot:** Stateless web servers, containerized microservices behind an ALB, Kubernetes worker nodes, CI/CD runners, and batch processing.
* **Mixed Instances Policy in ASG:**
  * Configure your ASG with **30% On-Demand Base Capacity** (guaranteeing baseline availability) and **70% Spot Instances** across multiple instance families (e.g., `c5.large`, `c6i.large`, `c6g.large`) to absorb capacity pools without interruption.

---

### Auto Scaling Groups (ASG) Engineering

An ASG maintains your application availability through self-healing and dynamic scaling.

#### 1. Core Configuration:
* **Launch Template:** Defines AMI ID, instance type, Security Groups, IAM Instance Profile, EBS storage, and User Data script.
* **VPC Subnets:** Always specify subnets across at least 2 or 3 Availability Zones.
* **Min / Desired / Max Capacity:** e.g., Min=2, Desired=4, Max=20.

#### 2. Scaling Policies:
1. **Target Tracking Scaling (Recommended):** Keeps a specific metric at a setpoint (e.g., *"Keep average CPU utilization at 60%"* or *"Keep ALB RequestCountPerTarget at 1,000"*). AWS calculates how many instances to add or remove automatically.
2. **Step Scaling:** Add 2 instances if CPU > 70%; add 5 instances if CPU > 90%.
3. **Scheduled Scaling:** Pre-warm the cluster every Friday at 4:00 PM for weekend shopping traffic.

#### 3. Health Checks: EC2 vs. ELB:
* **EC2 Status Check:** Only detects hardware/hypervisor failures. If NGINX crashes or the app throws 500 errors, the EC2 check says the instance is still healthy!
* **ELB Health Check (Crucial):** Pings your HTTP health check endpoint (`/healthz`). If the application freezes or throws errors, the load balancer reports `Unhealthy` to the ASG, which immediately terminates the broken instance and provisions a fresh one.

---

##### 3. Visual Flow Diagram: ASG Self-Healing & Scaling Lifecycle

```
[ Application Load Balancer ]
       |
       | Periodic HTTP GET /healthz
       v
[ EC2 Instance i-abc (App crashed, returns 500) ]
       |
       v
[ ALB marks instance UNHEALTHY ] ===> Notifies ===> [ Auto Scaling Group ]
                                                             |
                                      +----------------------+----------------------+
                                      |                                             |
                                      v                                             v
                           [ Terminate i-abc ]                          [ Launch Fresh i-xyz ]
                           (Drain connection 300s)                      (Execute User Data)
                                                                                    |
                                                                                    v
                                                                        [ Register to ALB ]
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **The Connection Draining (Deregistration Delay) Trap:**
  * When an ASG scales in or terminates an unhealthy instance, does it kill the machine instantly?
  * If it kills the instance instantly, any active customer purchasing an item experiences a dropped connection!
  * **The Fix:** The Target Group enforces a **Deregistration Delay** (default: 300 seconds). The ALB stops sending *new* requests to the terminating instance, but allows active in-flight requests 300 seconds to complete gracefully before the ASG sends the final `SIGTERM` / termination command.

---

##### 5. Quick Hands-On Check

Trigger a non-disruptive, rolling instance refresh across an entire Auto Scaling Group (e.g., after updating the Launch Template with a new AMI):

```bash
# Start an automated rolling instance refresh with 50% min healthy percentage
aws autoscaling start-instance-refresh \
    --auto-scaling-group-name production-web-asg \
    --preferences '{"MinHealthyPercentage": 50, "InstanceWarmup": 300}' \
    --region us-east-1
```
