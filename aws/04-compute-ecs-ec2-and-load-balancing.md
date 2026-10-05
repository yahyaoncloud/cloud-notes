# AWS Compute: EC2, Containerized ECS (Fargate) & Load Balancing

---

## 1. Elastic Compute Cloud (EC2) vs Elastic Container Service (ECS)

| Feature | Amazon EC2 | AWS ECS on Fargate |
| :--- | :--- | :--- |
| **Model** | Virtual Machine (IaaS) | Serverless Container Engine (CaaS) |
| **OS Management** | Customer responsible for patching, hardening, AMI lifecycle. | AWS manages underlying host OS and container runtime. |
| **Networking** | Dedicated ENI with Private IP and optional Public IP. | `awsvpc` network mode gives each Task a dedicated VPC ENI. |
| **Scaling** | Auto Scaling Groups (ASG) based on CPU/Memory/Target tracking. | ECS Service Auto Scaling based on Task CPU/Memory/ALB request count. |

---

## 2. Application Load Balancer (ALB) Routing Logic

Operating at **Layer 7**, ALBs evaluate HTTP/HTTPS request attributes to route traffic to specific Target Groups:
* **Host-Based Routing:** `api.example.com` &rarr; Target Group API, `app.example.com` &rarr; Target Group Web.
* **Path-Based Routing:** `/api/v1/*` &rarr; Microservice A, `/static/*` &rarr; S3 Bucket / CDN.
* **HTTP Header / Method Routing:** Method `POST` or Header `X-Custom-Auth` matching.
* **TLS Termination & Redirects:** Automatic HTTP (80) to HTTPS (443) redirect with AWS Certificate Manager (ACM).

---

## 3. Route 53 Routing Policies

* **Simple Routing:** Standard single-record lookup (round-robin for multiple IPs).
* **Weighted Routing:** Distribute traffic by percentage (e.g. 90% Production, 10% Canary deployment).
* **Latency-Based Routing:** Routes queries to the AWS region with lowest network round-trip time for the end user.
* **Failover Routing:** Active-Passive failover driven by Route 53 Health Checks.
* **Geolocation / Geoproximity:** Routes traffic based on user's geographic location or proximity with bias.
