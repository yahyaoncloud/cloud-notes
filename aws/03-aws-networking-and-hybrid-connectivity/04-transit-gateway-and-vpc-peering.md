# AWS Transit Gateway (TGW), VPC Peering & Hub-and-Spoke Routing

As an organization grows from 2 VPCs to 200 VPCs across multiple AWS accounts, managing network interconnects becomes a major engineering bottleneck. This guide contrasts point-to-point **VPC Peering** with **AWS Transit Gateway (TGW)**, demonstrates network segmentation using TGW route table domains, and explains how to solve the classic firewall **asymmetric routing** problem using Appliance Mode.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: Point-to-Point Phone Cables vs. The Central Switchboard
Imagine a town with 10 houses:
* **VPC Peering (Direct Point-to-Point Wires):** To let every neighbor call each other, you run a physical copper wire between every pair of houses.
  * For 4 houses, you need $\frac{4 \times 3}{2} = 6$ cables.
  * For 50 houses, you need $\frac{50 \times 49}{2} = \mathbf{1,225\ cables}$!
  * If House A wants to talk to House C, they cannot route their voice through House B's telephone (**Non-Transitive Routing**).
* **AWS Transit Gateway (The Central High-Speed Telecom Switchboard):** Every house runs exactly **one** fiber cable into the town's central telephone exchange tower. The tower has programmable routing tables that can connect any house to any other house, block calls between teenagers and party lines (**Route Domains / Segmentation**), and route external long-distance calls through a central satellite dish (**Centralized Egress / VPN**).

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Architectural Comparison: Peering vs. Transit Gateway

```
+------------------------------------+------------------------------------+
| VPC PEERING                        | AWS TRANSIT GATEWAY (TGW)          |
+------------------------------------+------------------------------------+
| Point-to-point 1:1 connection      | Centralized Hub-and-Spoke router   |
| Full mesh required (N*(N-1)/2)     | Linear attachments (1 per VPC)     |
| NON-TRANSITIVE routing             | FULL TRANSITIVE routing supported  |
| No bandwidth limits (line rate)    | Up to 50 Gbps burst per attachment |
| Zero hourly cost (pay data only)   | ~$0.05/hr/attachment + data fees   |
| No route tables inside peering     | Multiple isolated TGW route tables |
| Up to ~125 peering links per VPC   | Up to 5,000 VPC attachments per TGW|
+------------------------------------+------------------------------------+
```

---

### The Non-Transitive Peering Rule

> If VPC A is peered with VPC B, and VPC B is peered with VPC C, **traffic from VPC A CANNOT reach VPC C through VPC B!**

```
[ VPC A: 10.1.0.0/16 ] <====== Peering ======> [ VPC B: 10.2.0.0/16 ] <====== Peering ======> [ VPC C: 10.3.0.0/16 ]
       |                                                                                             |
       +----------------------------- CANNOT ROUTE TO EACH OTHER ------------------------------------+
```

To enable communication between A and C, you must establish an explicit third peering connection directly between A and C.

---

### Enterprise Hub-and-Spoke Architecture with Transit Gateway

With AWS Transit Gateway, you provision a single logical router in a Region and attach:
* Spoke VPCs (Development, Staging, Production)
* Shared Services VPC (Active Directory, CI/CD runners, Bastions)
* Centralized Egress VPC (Central NAT Gateways & Next-Gen Firewalls)
* On-Premises IPsec VPN & Direct Connect connections

```
                                +-----------------------------+
                                |     SHARED SERVICES VPC     |
                                |       (10.100.0.0/16)       |
                                +--------------+--------------+
                                               |
                                        (VPC Attachment)
                                               |
                                               v
    +---------------------------+   +-------------------+   +---------------------------+
    |      PRODUCTION VPC       |   |    AWS TRANSIT    |   |      DEVELOPMENT VPC      |
    |      (10.10.0.0/16)       |==>|      GATEWAY      |<==|      (10.20.0.0/16)       |
    +---------------------------+   +---------+---------+   +---------------------------+
                                               |
                                        (VPC Attachment)
                                               |
                                               v
                                +-----------------------------+
                                |    CENTRALIZED EGRESS VPC   |
                                |    (NAT Gateways & NGFWs)   |
                                +-----------------------------+
```

---

### Multi-Domain Isolation (TGW Route Tables)

Just like a VRF (Virtual Routing and Forwarding) instance in Cisco hardware, a Transit Gateway can hold **multiple isolated Route Tables**:

```
+----------------------------------------------------------------------------------------------------+
| 1. PRODUCTION TGW ROUTE TABLE                                                                      |
| - Routes to: Production VPC, Shared Services VPC, On-Premises DC.                                  |
| - DOES NOT ROUTE TO: Development VPC (Complete cryptographic and routing isolation!).              |
+----------------------------------------------------------------------------------------------------+
| 2. DEVELOPMENT TGW ROUTE TABLE                                                                     |
| - Routes to: Development VPC, Shared Services VPC, Central Egress.                                 |
| - Cannot reach Production VPC!                                                                     |
+----------------------------------------------------------------------------------------------------+
| 3. SHARED SERVICES TGW ROUTE TABLE                                                                 |
| - Routes to: ALL attached VPCs.                                                                    |
+----------------------------------------------------------------------------------------------------+
```

---

### The Firewall Asymmetric Routing Dilemma & Appliance Mode

When routing inter-VPC traffic through a centralized inspection VPC containing stateful Next-Gen Firewalls (e.g., Fortinet FortiGate, Palo Alto VM-Series):
1. **The Problem:** AWS VPC routing hashes packets based on a 5-tuple (Src IP, Dst IP, Src Port, Dst Port, Protocol).
2. Outbound traffic from VPC A to VPC B hits the firewall in **AZ-1a**. The firewall creates a session tracking table entry.
3. The return traffic from VPC B to VPC A might get routed by the Transit Gateway to the firewall instance in **AZ-1b**!
4. The firewall in AZ-1b has no record of the initial TCP handshake. It drops the return packet with `TCP RST` or `Out of state drop`!
5. **The Solution: TGW Appliance Mode.**
   * When enabled on the Inspection VPC attachment (`appliance-mode-support enable`), the Transit Gateway **guarantees symmetrical return routing**: return packets are forced through the exact same Availability Zone and firewall instance that processed the forward flow.

---

##### 3. Visual Flow Diagram: Appliance Mode Symmetry

```
WITHOUT APPLIANCE MODE (Asymmetric Routing Failure)
[VPC A] ---> [TGW] ---> [FW-1 in AZ-1a] ---> [VPC B]
                        (Session Created)
[VPC B] ---> [TGW] ---> [FW-2 in AZ-1b] ---> DROPPED! (No session state!)

WITH APPLIANCE MODE ENABLED (Guaranteed Symmetry)
[VPC A] ---> [TGW] ---> [FW-1 in AZ-1a] ---> [VPC B]
[VPC B] ---> [TGW] ===> FORCED BACK TO [FW-1 in AZ-1a] ===> [VPC A] (Success!)
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Can Transit Gateway peer across AWS Regions?"**
  * Yes! TGW Peering allows you to connect a Transit Gateway in `us-east-1` directly to a Transit Gateway in `eu-west-1` over the private AWS global fiber backbone with built-in encryption.

---

##### 5. Quick Hands-On Check

Enable Appliance Mode on your security inspection VPC attachment via the AWS CLI:

```bash
# Enable Appliance Mode on an existing Transit Gateway VPC Attachment
aws ec2 modify-transit-gateway-vpc-attachment \
    --transit-gateway-attachment-id tgw-attach-0123456789abcdef0 \
    --options ApplianceModeSupport=enable \
    --region us-east-1
```
