# GCP VPC Networking, Shared VPC & Hybrid Interconnect

Unlike AWS and Azure where VPCs and VNets are regional constructs, Google Cloud VPCs are **global** by default. Subnets within the VPC are **regional**.

---

## 1. GCP Global VPC Architecture

```text
Global VPC Network: "production-vpc" (Custom Mode)
├── us-central1 (Iowa)
│   ├── Subnet A: 10.10.1.0/24 (Compute VMs, GKE Nodes)
│   └── Secondary Ranges: Pods (10.100.0.0/16), Services (10.101.0.0/20)
├── europe-west1 (Belgium)
│   ├── Subnet B: 10.20.1.0/24
│   └── Cloud NAT Gateway + Cloud Router
└── asia-south1 (Mumbai)
    └── Subnet C: 10.30.1.0/24
```

### Key Differences vs AWS & Azure
* **Global Routing**: Resources in different regions communicate across Google's private fiber backbone using private IP addresses without needing VPNs or VPC peering.
* **Auto Mode vs. Custom Mode**: Always build with **Custom Mode** in production. Auto mode creates subnets in all regions using predefined `/20` CIDRs that conflict with enterprise networks.
* **Firewall Rules**: GCP firewalls are distributed stateful firewalls evaluated at the VM metadata layer (using network tags and service accounts).

---

## 2. Shared VPC (Multi-Project Enterprise Topology)

Shared VPC allows an organization to connect resources from multiple service projects to a common, centralized VPC network.

```text
Host Project: "net-core-hub"
└── Production Global VPC
    ├── Subnet-Apps (10.10.1.0/24) ──── Attached ───> Service Project A (Team Payments)
    └── Subnet-Data (10.10.2.0/24) ──── Attached ───> Service Project B (Team Analytics)
```

* **Separation of Duties**: Network Admins maintain IP allocation, routing, firewalls, and interconnects in the Host Project. Project Owners deploy VMs and GKE clusters in their respective Service Projects without network privileges.

---

## 3. Hybrid Connectivity Options

| Mechanism | Bandwidth | Encryption | Typical Use Case |
| :--- | :--- | :--- | :--- |
| **Cloud VPN (HA VPN)** | Up to 3 Gbps per tunnel (99.99% SLA with 2 tunnels) | IPsec (AES-GCM) over public Internet | Dev/Staging, Branch offices, Quick cloud migrations |
| **Dedicated Interconnect** | 10 Gbps or 100 Gbps circuits | Unencrypted (or MACsec at L2, Cloud Interconnect HA VPN) | High-volume production datacenter-to-GCP traffic |
| **Partner Interconnect** | 50 Mbps up to 10 Gbps | Standard L2/L3 circuit via telecom partner (Equinix, Megaport) | Datacenters not co-located in a Google colocation facility |

---

## 4. Terraform Shared VPC & Subnet Deployment

```hcl
resource "google_compute_shared_vpc_host_project" "host" {
  project = "yahya-net-host-prod"
}

resource "google_compute_shared_vpc_service_project" "service1" {
  host_project    = google_compute_shared_vpc_host_project.host.project
  service_project = "yahya-workload-prod"
}

resource "google_compute_network" "custom_vpc" {
  name                    = "prod-vpc"
  auto_create_subnetworks = false
  project                 = google_compute_shared_vpc_host_project.host.project
}

resource "google_compute_subnetwork" "subnet_us" {
  name          = "subnet-us-central1"
  ip_cidr_range = "10.10.0.0/20"
  region        = "us-central1"
  network       = google_compute_network.custom_vpc.id
  project       = google_compute_shared_vpc_host_project.host.project

  secondary_ip_range {
    range_name    = "gke-pods"
    ip_cidr_range = "10.100.0.0/16"
  }
}
```
