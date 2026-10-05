# Cloud Engineering Study Mindmap & Syllabus (SYL)

A hierarchical keyword-driven syllabus and curriculum map for mastering Multi-Cloud Architecture across **AWS**, **Azure**, **GCP**, and **Cloudflare**.

---

## 1. Multi-Cloud Global Overview

```mermaid
mindmap
  root((Cloud Engineering))
    AWS
      Networking
        VPC CIDR / Subnets
        Route Tables / IGW / NAT GW
        Transit Gateway / Peering
        Direct Connect / Site-to-Site VPN
        CloudFront / Route 53
      Identity and Governance
        Organizations / SCPs
        IAM Roles / Policies / Conditions
        Permission Boundaries
        KMS Envelope Encryption
      Compute and Containers
        EC2 / ASG / Launch Templates
        ALB / NLB / GWLB
        ECS Fargate
        EKS / Karpenter / CNI
      Storage and Databases
        S3 / Lifecycle / Replication
        EFS / EBS gp3
        Aurora / RDS Multi-AZ
        DynamoDB Single-Table
        ElastiCache Redis
      Serverless and Integration
        Lambda / VPC Cold Starts
        EventBridge / Event Bus
        SQS FIFO / SNS Fanout
        Step Functions
      Security and Observability
        AWS WAF / Shield
        GuardDuty / Security Hub
        CloudTrail / CloudWatch Logs
    Azure
      Governance and Identity
        Management Groups / Subscriptions
        Microsoft Entra ID / Conditional Access
        RBAC / PIM / Azure Policy
        Azure Landing Zones
      Networking
        Virtual Networks / Subnets
        NSGs / ASGs
        VNet Peering / Virtual WAN
        Private Endpoints / Private Link
        Azure Firewall / App Gateway
      Compute and Services
        Azure VMs / Scale Sets
        Azure App Services
        Azure Kubernetes Service (AKS)
        Azure Functions
      Storage and Databases
        Azure Blob / Tiering
        Azure SQL Database / Hyperscale
        Cosmos DB Multi-Region
      Security and Operations
        Azure Key Vault
        Microsoft Defender for Cloud
        Microsoft Sentinel (SIEM)
        Azure Monitor / Log Analytics
    GCP
      Hierarchy and Governance
        Organization / Folders / Projects
        Cloud IAM / Predefined Roles
        Service Accounts / Impersonation
        Workload Identity Federation
        Organization Policy Constraints
      Networking
        Global VPC (Custom Mode)
        Regional Subnets
        Shared VPC (Host & Service)
        Cloud Router / Cloud NAT
        Cloud Interconnect / HA VPN
        Cloud Armor / Cloud CDN
      Compute and Containers
        Compute Engine / MIGs
        GKE Standard vs Autopilot
        Cloud Run (Concurrency / Scale-to-Zero)
        Eventarc / Cloud Functions
      Storage and Data
        Cloud Storage (Multi/Dual-Region)
        BigQuery (Colossus / Dremel / Partitioning)
        Cloud SQL / Cloud Spanner (TrueTime)
      Security and Monitoring
        Security Command Center (SCC)
        Secret Manager / KMS
        VPC Service Controls (VPC-SC)
        Cloud Logging / Cloud Monitoring
    Cloudflare
      Edge and Caching
        Anycast DNS / DNSSEC
        Orange Cloud vs Grey Cloud
        Full (Strict) SSL/TLS
        Cache Rules / Stale-While-Revalidate
        Argo Smart Routing
      Zero Trust and SASE
        Cloudflare One
        Cloudflare Access (IdP / OIDC)
        Cloudflare Gateway (DNS / HTTP / DLP)
        WARP Client
        Remote Browser Isolation (RBI)
      Connectivity and Ingress
        Cloudflare Tunnel (cloudflared)
        Zero Inbound Ports
        Private CIDR Routing
        SSH / RDP over Access
      Compute and Edge Security
        Cloudflare Workers (V8 Isolates)
        KV / D1 / Durable Objects
        Cloudflare Pages
        WAF Managed Rules / Wirefilter
        Rate Limiting / Bot Management
```

---

## 2. Text Keyword Hierarchy

### I. Amazon Web Services (AWS)
* **Foundations**: Well-Architected Framework • Shared Responsibility Model • Multi-Account Architecture (Control Tower, AWS Organizations, Service Control Policies)
* **Networking**: VPC CIDR Planning • Public/Private Subnets • Route Tables • Internet Gateway (IGW) • Egress-Only IGW • NAT Gateway • VPC Peering • AWS Transit Gateway (TGW) • AWS Direct Connect (DX) • AWS Site-to-Site VPN • Route 53 (Latency, Geolocation, Failover routing) • CloudFront CDN
* **Identity & Access**: IAM Users/Groups/Roles • AssumeRole API • Trust Policies vs Permission Policies • ABAC vs RBAC • Permission Boundaries • IAM Access Analyzer • AWS KMS (Customer Managed Keys, Envelope Encryption) • AWS Secrets Manager
* **Compute & Orchestration**: EC2 Lifecycle (On-Demand, Spot, Reserved) • Auto Scaling Groups (ASG) • Target Tracking • Application Load Balancers (ALB) • Network Load Balancers (NLB) • ECS (Fargate, Task Definitions, Service Discovery) • EKS (Cluster Autoscaler, Karpenter, AWS VPC CNI, IRSA)
* **Storage & Databases**: S3 (Standard, IA, Glacier Flexible/Deep, Object Lock, Versioning, CORS) • EBS (gp3, io2 Block Express) • EFS • Amazon RDS (Multi-AZ, Read Replicas) • Aurora Serverless v2 • DynamoDB (Partition Keys, Sort Keys, GSI, LSI, DynamoDB Streams) • ElastiCache (Redis cluster mode)
* **Serverless & Messaging**: AWS Lambda (Exec environment, layers, concurrency limits, VPC ENI attachments) • Amazon SQS (Dead Letter Queues, FIFO deduplication) • Amazon SNS (Topic subscriptions, message filtering) • Amazon EventBridge (Event buses, schema registry) • AWS Step Functions (State machines, retry/catch)
* **Security & Observability**: AWS WAF (Web ACLs, rate-based rules) • AWS Shield Standard/Advanced • AWS GuardDuty • AWS Security Hub • CloudTrail (Management vs Data events) • CloudWatch (Metrics, Logs Insights, Alarms) • AWS X-Ray (Distributed tracing)

---

### II. Microsoft Azure
* **Governance & Identity**: Management Group Hierarchy • Subscriptions • Resource Groups • Microsoft Entra ID (Tenants, App Registrations, Service Principals) • Conditional Access Policies • Privileged Identity Management (PIM) • RBAC Built-in Roles • Azure Policy Definitions & Initiatives • Azure Landing Zones
* **Networking**: Azure Virtual Network (VNet) • Subnets & Subnet Delegation • Network Security Groups (NSGs) • Application Security Groups (ASGs) • VNet Peering (Global/Regional) • Azure Virtual WAN (vWAN) • Azure Route Server • User Defined Routes (UDR) • Azure Firewall • Azure Application Gateway (WAF v2) • Private Link & Private Endpoints
* **Compute & Services**: Azure Virtual Machines • Virtual Machine Scale Sets (VMSS) • Azure App Service (App Service Plans, Deployment Slots) • Azure Kubernetes Service (AKS) (Azure CNI vs Kubenet, Workload Identity, Node pools) • Azure Functions (Consumption, Premium)
* **Storage & Databases**: Azure Storage Accounts (Blob, File, Queue, Table, Hierarchical Namespace for Data Lake) • Storage Tiering (Hot, Cool, Cold, Archive) • Azure SQL Database (Single, Elastic Pool, Hyperscale) • Azure Cosmos DB (Multi-region writes, Consistency levels)
* **Security & Operations**: Azure Key Vault (Keys, Secrets, Certificates) • Microsoft Defender for Cloud • Microsoft Sentinel (SIEM/SOAR, Data Connectors, Analytic Rules) • Azure Monitor • Log Analytics Workspaces • Kusto Query Language (KQL)

---

### III. Google Cloud Platform (GCP)
* **Resource Hierarchy & Governance**: Organization Node • Folder Nesting • Projects • Cloud IAM (Primitive, Predefined, Custom Roles) • Service Accounts & Keyless Workload Identity • Organization Policy Constraints • Audit Logging
* **Networking**: Global VPC (Custom Mode) • Regional Subnets • Alias IP Ranges • Shared VPC (Host Projects & Service Projects) • Cloud Router • Cloud NAT (Manual vs Auto IP) • Cloud Interconnect (Dedicated & Partner) • HA VPN • Cloud Armor (WAF & DDoS) • Cloud CDN
* **Compute & Modern Apps**: Compute Engine (Custom Machine Types, Managed Instance Groups) • Google Kubernetes Engine (GKE) (Standard vs Autopilot, VPC-native clusters, Workload Identity) • Cloud Run (Serverless containers, concurrency tuning, min-instances) • Eventarc • Cloud Tasks
* **Data & Storage**: Cloud Storage (Standard, Nearline, Coldline, Archive, Object Lifecycle Management) • BigQuery (Storage Colossus, Compute Dremel, Table Partitioning, Clustering, BI Engine) • Cloud SQL (Private IP, Read replicas) • Cloud Spanner (Global ACID transactions, TrueTime API)
* **Security & Operations**: Security Command Center (SCC) • Secret Manager • Cloud KMS • VPC Service Controls (Perimeters, Access Context Manager) • Cloud Logging • Cloud Monitoring • Cloud Trace

---

### IV. Cloudflare Edge Architecture
* **Edge Routing & Caching**: Global Anycast DNS Network • DNSSEC • Proxied Mode (Orange Cloud) vs DNS-Only (Grey Cloud) • Full (Strict) SSL/TLS with Origin CA • Cache Rules & Page Rules • Tiered Cache • Cache Reserve • Argo Smart Routing
* **Zero Trust & SASE**: Cloudflare One Architecture • Cloudflare Access (Identity Provider Integration, Contextual Policies, mTLS) • Cloudflare Gateway (DNS Filtering, HTTP Inspection, DLP) • WARP Client (WireGuard protocol) • Remote Browser Isolation (RBI)
* **Ingress & Hybrid Cloud**: Cloudflare Tunnel (`cloudflared`) • Zero Open Inbound Ports • Split-Tunnel Routing • Private IP Routing over WARP • Bastionless SSH & RDP via Access
* **Edge Compute & Security**: Cloudflare Workers (Google V8 Isolates) • Workers KV (Eventually consistent edge key-value) • D1 SQL Database • Durable Objects (Strong consistency) • Cloudflare Pages • WAF Managed Rule Sets • Wirefilter Expression Engine • Rate Limiting Rules • Bot Management
