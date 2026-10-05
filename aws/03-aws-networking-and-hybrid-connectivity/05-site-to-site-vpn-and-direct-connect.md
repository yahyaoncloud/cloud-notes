# Hybrid Cloud Connectivity: AWS Site-to-Site VPN & Direct Connect (DX)

Enterprises rarely run 100% of their operations exclusively in AWS on day one. Most operate in a hybrid model where cloud VPCs connect to on-premises data centers, branch offices, and co-location facilities. This guide deconstructs **IPsec VPN engineering**, **AWS Direct Connect (DX)**, and production failover architectures.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Armored Convoy vs. The Private Underground Railway
Imagine needing to transport high-value cargo between your corporate headquarters and a regional vault:
* **AWS Site-to-Site VPN (The Armored Convoy on Public Highways):**
  * You load your cargo into heavily armored trucks with bulletproof glass and armed guards (**IPsec Encryption: AES-256**).
  * You drive on the public highway (**The Public Internet**).
  * It is cheap and fast to set up (you just buy the trucks), but you are at the mercy of public traffic jams, weather, and potholes (**Jitter, Variable Latency, and Packet Drops**).
* **AWS Direct Connect (The Dedicated Private Underground Rail Line):**
  * You hire a construction company to lay a private, dedicated physical steel rail line connecting your building directly into AWS's loading dock (**Physical Fiber Optic Cross-Connect**).
  * No public cars are allowed on this track.
  * It costs significantly more to construct and takes weeks to provision, but travel time is 100% consistent, speed is massive (100 Gbps), and cargo never encounters public traffic.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### How IPsec VPN Works Deep Dive

An IPsec (Internet Protocol Security) tunnel is not a single protocol; it is a suite of cryptographic protocols operating in two distinct phases:

```
[ ON-PREMISES ROUTER / CGW ]                              [ AWS TRANSIT GATEWAY / VGW ]
             |                                                          |
             | === PHASE 1: IKE SA (Negotiate Management Channel) =====> |
             |     - Authenticates peers (Pre-Shared Key / RSA cert)    |
             |     - Diffie-Hellman Key Exchange derives secret         |
             |     - Establishes secure control plane tunnel (UDP 500)  |
             |                                                          |
             | === PHASE 2: IPsec SA (Negotiate Data Tunnels) =========> |
             |     - Negotiates Encapsulating Security Payload (ESP)    |
             |     - Creates two unidirectional tunnels (UDP 4500)      |
             |     - Derives session keys for AES-256-GCM encryption    |
             |                                                          |
             | <====== [ BGP Peering / Encrypted Traffic Flows ] ======> |
```

#### 1. Phase 1 (IKEv1 / IKEv2 SA):
* **Purpose:** Establish a secure, authenticated channel over which the routers can safely talk to negotiate traffic rules.
* **Ports:** UDP Port 500 (or UDP 443/4500 if NAT-Traversal is used).
* **Authentication:** Pre-Shared Key (PSK) or X.509 Certificates.

#### 2. Phase 2 (IPsec SA - Quick Mode):
* **Purpose:** Create the actual data tunnels that encrypt and encapsulate payload packets using **ESP (Encapsulating Security Payload, IP Protocol 50)**.
* **Dead Peer Detection (DPD):** Periodic keepalives (every 10-30s). If AWS stops receiving DPD hellos from the on-premises router, it tears down the route and fails over to Tunnel 2.

---

### AWS Site-to-Site VPN Topologies

When you create an AWS Site-to-Site VPN:
* AWS **always provisions TWO tunnels** per VPN connection across distinct Availability Zones for redundancy.
* **Customer Gateway (CGW):** The logical representation of your on-premises firewall/router (its public IP and BGP ASN).
* **Virtual Private Gateway (VGW):** Legacy anchor attached to a single VPC (no transitive routing).
* **Transit Gateway (TGW):** Modern enterprise anchor; allows on-premises to route to hundreds of attached VPCs.
* **Throughput Limit:** Each IPsec tunnel has a hard bandwidth cap of **1.25 Gbps**.
* **Scaling Beyond 1.25 Gbps (ECMP):** By enabling Equal-Cost Multi-Path (ECMP) routing on the Transit Gateway, you can aggregate multiple VPN tunnels in parallel to achieve 5 Gbps to 10 Gbps of aggregate encrypted bandwidth!

---

### AWS Direct Connect (DX) Deep Dive

AWS Direct Connect bypasses the public internet completely by linking your network directly to an AWS Direct Connect Location via a 1 Gbps, 10 Gbps, or 100 Gbps dedicated fiber line.

#### The 3 Virtual Interface (VIF) Types:
1. **Private VIF:** Connects your on-premises network directly to a single VPC (or a Direct Connect Gateway linked to multiple VPCs). Uses private RFC 1918 IP addresses.
2. **Public VIF:** Allows your on-premises data center to access **all global AWS public endpoints** (Amazon S3, DynamoDB, CloudFront, Route 53) over the private physical DX cable without using your commercial ISP!
3. **Transit VIF:** Connects your dedicated physical line directly to an **AWS Transit Gateway**, giving you 1-hop access to thousands of VPCs.

---

### Production Hybrid Failover Architecture: DX + Backup VPN

Enterprises use Direct Connect as their primary high-speed pipeline and an IPsec VPN as an automated backup.

```
                                  [ AWS TRANSIT GATEWAY ]
                                       /           \
                 (Transit VIF)        /             \ (IPsec Tunnel)
                                     /               \
                       [ AWS Direct Connect ]    [ Public Internet ]
                                     \               /
                                      \             /
                                [ ON-PREMISES DATA CENTER ]
                                [ BGP Edge Router (AS 65000) ]
```

#### How BGP Guarantees Deterministic Failover:
To ensure traffic always uses Direct Connect during normal operations and only flips to the VPN if the physical fiber is cut:

1. **Inbound to On-Premises (AWS -> DC):**
   * On-premises advertises its prefixes over both BGP sessions.
   * On the VPN BGP session, on-prem uses **BGP AS-Path Prepending** (e.g. prepending `65000 65000 65000`).
   * AWS BGP path selection prefers the shorter AS-path (Direct Connect).
2. **Outbound to AWS (DC -> AWS):**
   * On-prem router assigns a higher **BGP Local Preference** (`local-preference 200`) to routes received over Direct Connect and a lower preference (`local-preference 100`) to routes over the VPN.

---

##### 3. Visual Flow Diagram: BGP Path Preference Failover

```
[AWS Transit Gateway]
    |
    |---- (Route: 10.0.0.0/8 via DX  | AS-Path: 65000)        <=== PREFERRED (Shorter Path)
    |
    `---- (Route: 10.0.0.0/8 via VPN | AS-Path: 65000 65000 65000) <-- BACKUP (Prepended)
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Does Direct Connect encrypt traffic by default?"**
  * *The Dangerous Trap:* **NO!**
  * Direct Connect is a private dedicated physical connection, but by default, data travels over the fiber in **plaintext Ethernet frames**.
  * If your company requires cryptographic encryption in transit for regulatory compliance (HIPAA, PCI-DSS), you must either:
    1. Deploy **MACsec (802.1AE)** point-to-point layer 2 encryption on dedicated 10G/100G DX ports.
    2. Run an **IPsec VPN tunnel on top of the Direct Connect Transit VIF**!

---

##### 5. Quick Hands-On Check

Verify the operational status of your AWS Site-to-Site VPN tunnels via the CLI:

```bash
# Check if both IPsec tunnels are UP and verify BGP routing status
aws ec2 describe-vpn-connections \
    --vpn-connection-ids vpn-0123456789abcdef0 \
    --query "VpnConnections[*].VgwTelemetry[*].{TunnelIP:OutsideIpAddress,Status:Status,BgpStatus:StatusMessage}" \
    --output table
```
