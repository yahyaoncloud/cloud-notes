# Production 3-Tier Enterprise AWS Architecture: Design & Interview Defense

The 3-tier web architecture across multiple Availability Zones is the gold standard whiteboard design scenario in cloud engineering interviews. This guide walks through the complete end-to-end design, component justifications, traffic flows, failure scenarios, and interview defense strategies.

---

#### The Production 3-Tier Architecture Blueprint

```
                                  [ PUBLIC INTERNET ]
                                           |
                                           v
                             [ Amazon Route 53 (DNS) ]
                                           |
                                           v
                        [ AWS WAF + Amazon CloudFront (CDN) ]
                                           |
===========================================|======================================================
AWS REGION: us-east-1                      |
VPC: 10.100.0.0/16                         | (HTTPS / TLS 1.3)
-------------------------------------------|------------------------------------------------------
TIER 1: PRESENTATION / PUBLIC SUBNETS      v
+------------------------------------------------------------------------------------------------+
|  Availability Zone A (us-east-1a)              |  Availability Zone B (us-east-1b)             |
|  Public Subnet A: 10.100.1.0/24                |  Public Subnet B: 10.100.2.0/24               |
|                                                |                                               |
|  [ Internet-Facing Application Load Balancer ] |  [ Internet-Facing Application Load Balancer ]|
|  [ NAT Gateway A (Elastic IP: 54.1.1.10) ]     |  [ NAT Gateway B (Elastic IP: 54.2.2.20) ]    |
+------------------------------------------------+-----------------------------------------------+
                                                 |
         +---------------------------------------+---------------------------------------+
         | (Private HTTP: Port 8080)                                                     |
         v                                                                               v
TIER 2: APPLICATION / PRIVATE APP SUBNETS
+------------------------------------------------+-----------------------------------------------+
|  App Subnet A: 10.100.10.0/24                  |  App Subnet B: 10.100.11.0/24                 |
|                                                |                                               |
|  [ EC2 Auto Scaling Group / ECS Fargate Tasks] |  [ EC2 Auto Scaling Group / ECS Fargate Tasks]|
|  - Managed via AWS Systems Manager (SSM)       |  - Managed via AWS Systems Manager (SSM)      |
|  - Egress to Internet via NAT Gateway A        |  - Egress to Internet via NAT Gateway B       |
+------------------------------------------------+-----------------------------------------------+
                                                 |
         +---------------------------------------+---------------------------------------+
         | (Private PostgreSQL Port 5432 / Redis Port 6379)                              |
         v                                                                               v
TIER 3: DATA / PRIVATE ISOLATED DATABASE SUBNETS (ZERO OUTBOUND INTERNET ACCESS!)
+------------------------------------------------+-----------------------------------------------+
|  Data Subnet A: 10.100.20.0/24                 |  Data Subnet B: 10.100.21.0/24                |
|                                                |                                               |
|  [ Amazon Aurora PostgreSQL - Primary Writer ] |  [ Amazon Aurora PostgreSQL - Read Replica ]  |
|  [ Amazon ElastiCache Redis - Primary Node ]   |  [ Amazon ElastiCache Redis - Replica Node ]  |
|                                                |                                               |
|  <================ Synchronous 6-Way Shared Storage Volume Replication ================>      |
+------------------------------------------------------------------------------------------------+
==================================================================================================
```

---

### Step-by-Step Layer Justification for Interviews

When presenting this architecture to an interviewer, walk through each tier chronologically:

#### 1. Edge & DNS Tier (Route 53, CloudFront, WAF)
* **Route 53:** Provides DNS resolution with Health Check Failover and Alias records pointing to CloudFront.
* **Amazon CloudFront:** Edge CDN terminating TLS close to end users, caching static web assets, and routing API traffic over the AWS private global fiber network.
* **AWS WAF:** Attached to CloudFront. Blocks SQL injection, cross-site scripting (XSS), and applies rate-limiting rules (e.g. max 2,000 requests per 5 minutes per IP) before traffic enters the VPC.

#### 2. Tier 1: Presentation & Public Subnets
* **Internet-Facing ALB:** Distributed across AZ-1a and AZ-1b. Inspects HTTP headers and terminates SSL/TLS using certificates from AWS Certificate Manager (ACM).
* **Multi-AZ NAT Gateways:** Each AZ has its own dedicated NAT Gateway. If AZ-1a fails, AZ-1b retains full, independent internet egress.

#### 3. Tier 2: Application & Private Subnets
* **Compute Layer:** Containerized workloads running on **AWS Fargate** or **EC2 Auto Scaling Groups** (mixed 30% On-Demand, 70% Spot for cost efficiency).
* **Security Groups:** Ingress permitted **only** from the ALB Security Group on port 8080. Zero direct internet access.
* **Zero SSH Keys:** Admin access is handled exclusively via **AWS Systems Manager (SSM) Session Manager** with MFA and full session logging to CloudWatch.

#### 4. Tier 3: Data & Isolated Database Subnets
* **Amazon Aurora PostgreSQL:** High availability with automated failover in under 30 seconds.
* **Amazon ElastiCache Redis:** In-memory caching tier reducing database query load by 85%.
* **Strict Network Isolation:** Database subnets have **no default route to the internet** (`0.0.0.0/0` is absent). Even if malicious code compromises an application container, the database cannot be exfiltrated to external IPs.

---

### How to Defend This Design Against Interviewer "Stress Questions"

#### Question 1: "What happens if an entire AWS Availability Zone experiences a blackout?"
* **Defense:**
  1. The Route 53 and CloudFront layers automatically stop routing traffic to the impacted AZ.
  2. The Application Load Balancer health check detects that target nodes in AZ-1a are unresponsive and shifts 100% of traffic to healthy tasks in AZ-1b.
  3. The Auto Scaling Group senses capacity drop and spawns replacement tasks in AZ-1b.
  4. If the primary database writer was in AZ-1a, Aurora initiates an automated failover to the reader replica in AZ-1b, promoting it to primary in under 30 seconds. RPO = 0 (zero transaction loss), RTO < 30 seconds.

#### Question 2: "Our security compliance auditor demands that application servers cannot access the public internet, but they must download Docker images and talk to S3. How do you configure this?"
* **Defense:**
  1. Eliminate the NAT Gateways entirely from the route tables.
  2. Deploy **VPC Endpoints**:
     * An **S3 Gateway Endpoint** (free) attached to the private route tables.
     * **Interface Endpoints (AWS PrivateLink)** for Amazon ECR (`com.amazonaws.region.ecr.api` and `dkr`), AWS Secrets Manager, and AWS CloudWatch Logs.
  3. Container instances download images, stream logs, and access data completely over private AWS internal networking.
