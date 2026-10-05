# AWS Virtual Private Cloud (VPC) Architecture, Subnetting & Route Tables

Networking is the nervous system of cloud architecture. If your network design is flawed, no amount of auto-scaling, containerization, or Kubernetes will save your production environment from security breaches or routing failures. This guide deconstructs AWS VPC engineering, IP address planning, subnet classification, and route table mechanics.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Corporate Office Complex
Imagine building a secure, high-tech corporate campus:
* The **VPC (e.g., `10.100.0.0/16`)** is the perimeter fence surrounding the entire parcel of land. Nothing enters or leaves without passing an official checkpoint.
* The **Subnets** are individual buildings inside the campus:
  * The **Public Subnet (Lobby / Reception Desk)** is where visitors arrive. The front doors are wide open to the public street (**Internet Gateway**).
  * The **Private Application Subnet (Executive Offices / Engineering R&D)** is in a secure inner building. Outsiders cannot walk in directly; visitors must be escorted by an internal receptionist (**Load Balancer**). Employees can send letters outside through an outgoing-only postal clerk (**NAT Gateway**).
  * The **Isolated Database Subnet (The Bank Vault / Cash Room)** has no windows and no doors to the street. Only authorized financial staff from the inner building can access the vault through a secure biometric hallway (**Local VPC Route**).
* The **Route Table** is the campus security signage posted at every hallway intersection directing where traffic is allowed to turn.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### VPC IP Planning & CIDR Mathematics

When provisioning a VPC, you assign an IPv4 Classless Inter-Domain Routing (CIDR) block. The standard enterprise sizing is a `/16` (providing $2^{16} = 65,536$ total IP addresses).

```
VPC CIDR: 10.100.0.0/16 (65,536 Total IPs)
|
+---> Availability Zone us-east-1a:
|     |-- Public Subnet:       10.100.1.0/24   (256 IPs)  -> ALB, NAT Gateway A
|     |-- Private App Subnet:  10.100.10.0/24  (256 IPs)  -> EC2 / ECS / EKS Nodes
|     `-- Private Data Subnet: 10.100.20.0/24  (256 IPs)  -> RDS Aurora Primary
|
`---> Availability Zone us-east-1b:
      |-- Public Subnet:       10.100.2.0/24   (256 IPs)  -> ALB, NAT Gateway B
      |-- Private App Subnet:  10.100.11.0/24  (256 IPs)  -> EC2 / ECS / EKS Nodes
      `-- Private Data Subnet: 10.100.21.0/24  (256 IPs)  -> RDS Aurora Standby
```

---

### The 5 Reserved IP Addresses in Every AWS Subnet

In traditional on-premises networking, only 2 IP addresses are reserved in any subnet (the Network Address and the Broadcast Address).
**In AWS, exactly 5 IP addresses are reserved in every single subnet.**
If you create a `/24` subnet (256 theoretical IPs), you have exactly **251 usable IPs**:

Taking subnet `10.100.1.0/24` as the example:
1. `10.100.1.0`: **Network Address** (the base identifier of the subnet).
2. `10.100.1.1`: **VPC Router** (the default gateway internal address).
3. `10.100.1.2`: **AWS DNS Resolver** (Amazon Provided DNS / Route 53 Resolver at the base $+ 2$).
4. `10.100.1.3`: **Future Use** (reserved by AWS for internal expansion).
5. `10.100.1.255`: **Network Broadcast Address** (AWS does not support broadcast; all traffic is unicast).

*Interview Tip:* If an interviewer asks: *"What is the smallest subnet you can create in an AWS VPC?"*
The answer is **/28** (16 IPs total minus 5 AWS reserved = 11 usable IP addresses).

---

### The 3 Subnet Archetypes & Route Table Associations

A subnet is defined as "Public" or "Private" **solely based on its attached Route Table**, not by any magical property of the subnet itself!

```
+----------------------------------------------------------------------------------------------------+
| 1. PUBLIC SUBNET ROUTE TABLE                                                                       |
| Destination          Target                  Explanation                                           |
| 10.100.0.0/16        local                   Internal VPC east-west traffic stays inside VPC       |
| 0.0.0.0/0            igw-0123456789          Default route points to an Internet Gateway (IGW)     |
+----------------------------------------------------------------------------------------------------+
| 2. PRIVATE APPLICATION SUBNET ROUTE TABLE                                                          |
| Destination          Target                  Explanation                                           |
| 10.100.0.0/16        local                   Internal VPC traffic stays inside VPC                 |
| 0.0.0.0/0            nat-0123456789          Outbound-only internet egress via NAT Gateway         |
+----------------------------------------------------------------------------------------------------+
| 3. PRIVATE DATABASE / ISOLATED SUBNET ROUTE TABLE                                                  |
| Destination          Target                  Explanation                                           |
| 10.100.0.0/16        local                   Internal VPC traffic ONLY. Zero internet route!       |
+----------------------------------------------------------------------------------------------------+
```

---

### The Internet Gateway (IGW) Under the Hood

An Internet Gateway is **not** a physical router, a bottleneck, or a single point of failure.
* It is a software-defined, horizontally scaled, redundant VPC component managed by the AWS hypervisor.
* It performs **1-to-1 1:1 Network Address Translation (NAT)** between an EC2 instance's private IPv4 address and its assigned Public Elastic IP.
* It imposes **zero bandwidth constraints** (bandwidth scales with instance network interfaces).

---

##### 3. Visual Flow Diagram: VPC Traffic Routing Logic

```
   [ INTERNET ]
        ^
        | (Public IPv4)
        v
+-------+------------------------------------------------------------------------------------+
| [Internet Gateway (IGW)]                                                                   |
|       ^                                                                                    |
|       | (0.0.0.0/0 Route)                                                                  |
|   [Public Subnet 10.100.1.0/24]                                                            |
|   +---------------------------------------------------+                                    |
|   | [Internet-Facing ALB]                             |                                    |
|   | [NAT Gateway (Elastic IP: 54.210.1.10)]           |                                    |
|   +---------------------------------------------------+                                    |
|       ^ (Forward HTTP requests)       ^ (Egress outbound for package downloads)            |
|       |                               |                                                    |
|   [Private Application Subnet 10.100.10.0/24]                                              |
|   +---------------------------------------------------+                                    |
|   | [ECS Task / EC2 Instance (10.100.10.45)]          |                                    |
|   +---------------------------------------------------+                                    |
|       ^ (DB Queries: Port 5432)                                                            |
|       |                                                                                    |
|   [Private Isolated Database Subnet 10.100.20.0/24]                                        |
|   +---------------------------------------------------+                                    |
|   | [RDS Aurora PostgreSQL (10.100.20.80)]           | (ZERO Outbound Routes!)            |
|   +---------------------------------------------------+                                    |
+--------------------------------------------------------------------------------------------+
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"I gave my EC2 instance a Public IP address, but I still cannot reach the internet. Why?"**
  * *The Trap:* Giving an EC2 instance a public IP address does **not** automatically make it reachable.
  * To reach the internet:
    1. The instance must have a Public IP (or Elastic IP).
    2. The instance's subnet route table must have a default route `0.0.0.0/0 -> igw-xxxx`.
    3. The attached Security Group must permit outbound traffic (and inbound if receiving requests).
    4. The subnet NACL must permit traffic in both directions.
    *If any one of these four pillars is missing, connectivity fails.*

---

##### 5. Quick Hands-On Check

Query the route tables associated with your VPC and inspect their default routes using the AWS CLI:

```bash
# Describe all route tables in a VPC and inspect route destinations and targets
aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
    --query "RouteTables[*].{RouteTableId:RouteTableId,Routes:Routes[*].{Dest:DestinationCidrBlock,Target:GatewayId}}" \
    --output json
```
