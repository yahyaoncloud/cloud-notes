# Azure Compute: Virtual Machines, AKS & Application Gateway

---

## 1. Compute Services Comparison

* **Azure Virtual Machines (VM):** IaaS compute with full root/administrative access, Availability Sets (99.95% SLA) and Availability Zones (99.99% SLA).
* **Virtual Machine Scale Sets (VMSS):** Auto-scaling pool of identical VMs based on metrics or schedule.
* **Azure Kubernetes Service (AKS):** Managed Kubernetes control plane (free), customer manages worker node pools (VMSS backed) with Azure CNI or Kubenet networking.

---

## 2. Azure Application Gateway (WAF v2)

Operating at **Layer 7**, Application Gateway provides advanced HTTP/S load balancing, SSL termination, and integrated Web Application Firewall (OWASP CRS 3.2):
* **URL Path-Based Routing:** `/images/*` &rarr; Pool Images, `/video/*` &rarr; Pool Video.
* **Multi-Site Hosting:** Routing traffic for multiple domains (`contoso.com`, `fabrikam.com`) on the same public IP.
* **Cookie-Based Affinity:** Sticky sessions binding user requests to the same backend server.
* **Connection Draining:** Graceful removal of backend instances during updates.
