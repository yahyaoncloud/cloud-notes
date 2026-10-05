# Cloud Notes & Architecture Handbook ☁️

A comprehensive, production-tested reference for Multi-Cloud Engineering, Network Topology, Security Governance, and Infrastructure Systems across **AWS**, **Microsoft Azure**, **Google Cloud Platform (GCP)**, and **Cloudflare**.

---

## 📂 Repository Structure

```text
cloud-notes/
├── aws/           # Amazon Web Services core architectures, VPC, EKS, Serverless, IAM
├── azure/         # Microsoft Azure Entra ID, VNets, Hub-Spoke, AKS, Sentinel
├── gcp/           # Google Cloud Platform Global VPC, Shared VPC, GKE, Cloud Run, BigQuery
└── cloudflare/    # Cloudflare Zero Trust, Tunnels (cloudflared), CDN, Workers & WAF
```

---

## 🚀 Sections & Cloud Providers

### 1. [Amazon Web Services (AWS)](./aws/)
* **Multi-Tier Production VPC**: Subnet segmentation, NAT Gateways, Transit Gateway peering, Direct Connect.
* **Compute & Containers**: EC2 lifecycle, Auto-scaling, ECS (Fargate), Amazon EKS production setups.
* **Identity & Security**: IAM least privilege, Permission Boundaries, SCPs, AWS WAF, GuardDuty, KMS.
* **Storage & Databases**: S3 security & lifecycle policies, Aurora Multi-AZ, DynamoDB single-table design.
* **Serverless**: AWS Lambda internals, VPC cold starts, SQS/SNS/EventBridge decoupled architectures.

### 2. [Microsoft Azure](./azure/)
* **Enterprise Governance**: Microsoft Entra ID (Azure AD), Conditional Access, Management Groups, Azure Policy.
* **Network Topology**: Virtual Network (VNet) design, Hub-and-Spoke, Virtual WAN, Private Endpoints, NSGs.
* **Cloud Services**: Azure VMs, Virtual Machine Scale Sets (VMSS), App Services, Azure Kubernetes Service (AKS).
* **Security & Operations**: Azure Key Vault, Microsoft Defender for Cloud, Sentinel SIEM, Log Analytics.

### 3. [Google Cloud Platform (GCP)](./gcp/)
* **Resource Hierarchy**: Organization, Folders, Projects, Cloud IAM, Workload Identity Federation.
* **Global Networking**: Global VPCs, Subnets, Shared VPC (Host & Service projects), Cloud Router, Cloud NAT.
* **Modern Compute**: GKE Autopilot vs. Standard, Cloud Run serverless containers, MIGs.
* **Data & Analytics**: Cloud Storage classes, BigQuery architecture & query optimization, Cloud Spanner.

### 4. [Cloudflare](./cloudflare/)
* **Traffic & Edge Performance**: Global Anycast DNS, SSL/TLS Full (Strict), Tiered Cache, Cache Rules.
* **Zero Trust & SASE**: Cloudflare Access, Gateway inspection, WARP client, Remote Browser Isolation.
* **Cloudflare Tunnels (`cloudflared`)**: Zero-open-port ingress architecture, Private IP routing, SSH access.
* **Serverless & Edge Security**: Workers (V8 Isolates), Pages, KV, D1, Web Application Firewall (WAF).

---

## 🛠️ Maintained By
**Yahya** ([@yahyaoncloud](https://github.com/yahyaoncloud))
