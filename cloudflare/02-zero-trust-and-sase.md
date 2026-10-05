# Cloudflare Zero Trust & SASE Architecture

Cloudflare Zero Trust (part of Cloudflare One) replaces legacy VPN concentrators with identity-aware, context-driven access control at the edge.

---

## 1. Zero Trust Architectural Model

```text
Remote Engineer / User (WARP Client or Browser)
      │
      ▼
Cloudflare Edge PoP
      ├── 1. Identity Verification (Okta, Azure Entra ID, Google Workspace)
      ├── 2. Device Posture Check (OS version, Disk encryption, CrowdStrike agent)
      ├── 3. Contextual Evaluation (Country, IP range, Session time)
      │
      └── [PASSED] ──── Encrypted Tunnel ────> Internal Application / Server
```

---

## 2. Core Pillars of Cloudflare One

| Component | Functionality | Replaces |
| :--- | :--- | :--- |
| **Cloudflare Access** | Identity-aware proxy for web apps, SSH, and RDP. Injects user identity headers (`Cf-Access-Authenticated-User-Email`). | Bastion hosts, Open SSH ports, Cisco AnyConnect / OpenVPN |
| **Cloudflare Gateway** | Forward proxy inspecting DNS, HTTP, and network traffic. Blocks malware, phishing, and enforces Data Loss Prevention (DLP). | Next-Gen Firewalls (NGFW), on-prem web gateways |
| **WARP Client** | Lightweight WireGuard-based endpoint client routing device traffic to Cloudflare Zero Trust edge. | Full-tunnel legacy VPN clients |
| **Browser Isolation (RBI)** | Renders untrusted web code in isolated Cloudflare cloud containers and streams vector graphics to user browser. | Traditional local browser exposure to drive-by malware |

---

## 3. SSH Access Policy Configuration

Access policies allow defining granular rules per role:
* **Action: Allow**
  * Include: Emails ending in `@yahyaoncloud.com`
  * Require: Multi-Factor Authentication (MFA)
  * Require: Device Posture = Corporate Managed Device
