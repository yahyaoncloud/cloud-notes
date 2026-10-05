# AWS Interconnectivity: Transit Gateway (TGW), VPC Peering & IPsec VPN

---

## 1. VPC Peering vs AWS Transit Gateway

| Feature | VPC Peering | AWS Transit Gateway (TGW) |
| :--- | :--- | :--- |
| **Topology** | Full Mesh point-to-point connections ($\frac{N(N-1)}{2}$ links). | Centralized Hub-and-Spoke router. |
| **Transitive Routing** | **Not supported** (A &harr; B and B &harr; C does NOT allow A &harr; C). | **Fully supported** across attached VPCs, VPNs, and Direct Connect. |
| **Bandwidth** | Uncapped (No throughput bottleneck). | Up to 50 Gbps burst per VPC attachment. |
| **Scale** | Complex management beyond 10+ VPCs. | Scales up to 5,000 VPC attachments per region. |
| **Route Table Domains**| Per-VPC route tables only. | Multiple TGW Route Tables for domain isolation (e.g. Prod, Dev, Shared Services). |

---

## 2. Hub-and-Spoke Transit Architecture

```text
       [Shared Services VPC] (10.100.0.0/16)
               | (Attachment)
     +---------v---------+
     |   AWS Transit     | <==== IPsec VPN (BGP) ====> [On-Premises Data Center]
     |     Gateway       |
     +---------+---------+
          /         \
         v           v
  [Prod VPC]      [Dev VPC]
(10.101.0.0/16) (10.102.0.0/16)
```

---

## 3. Site-to-Site IPsec VPN Mechanics

* **Phase 1 (IKE SA):** Authenticates endpoints via Pre-Shared Key (PSK) or X.509 Certificate, establishes Diffie-Hellman (DH Group 14/19/20) shared secret, negotiates AES-256 encryption.
* **Phase 2 (IPsec SA):** Negotiates traffic selectors, establishes encryption/integrity transforms (ESP with AES-GCM or AES-CBC + SHA-256), enables Perfect Forward Secrecy (PFS).
* **Routing:** Dynamic BGP routing (ASN 64512–65534) enables automated route exchange and dual-tunnel failover.
