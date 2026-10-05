# Azure Virtual Network (VNet), Subnets & Network Security Groups (NSGs)

---

## 1. Azure VNet Architecture & IP Addressing

Azure Virtual Network (VNet) provides private network isolation in Azure:
* **Reserved IPs:** Azure reserves the **first 4 and last 1 IP address** in every subnet:
  * `.0`: Network Address
  * `.1`: Default Gateway
  * `.2`, `.3`: Azure DNS mapping / internal routing
  * `.255`: Broadcast Address
* **Subnet Delegation:** Assigns exclusive control of a subnet to specific Azure PaaS services (e.g. `Microsoft.Web/serverFarms` for App Service VNet integration, `Microsoft.ContainerInstance/containerGroups`).

---

## 2. Network Security Groups (NSG) vs Application Security Groups (ASG)

| Feature | Network Security Group (NSG) | Application Security Group (ASG) |
| :--- | :--- | :--- |
| **Layer** | L3/L4 Stateful Packet Filter | Logical Grouping of NICs for NSG Rules |
| **Rule Structure** | Priority (`100–4096`), Source IP/Port, Dest IP/Port, Protocol, Action. | Applied as Source/Destination in NSG rules. |
| **Attachment** | Attached to Subnet OR Network Interface (NIC). | Attached directly to VM Network Interfaces (NICs). |
| **Default Rules** | AllowVNetInBound (Priority 65000), AllowAzureLoadBalancerInBound (65001), DenyAllInBound (65500). | N/A |

```text
[Internet] -> [NSG on Subnet (Priority 100: Allow 443 to ASG-Web)]
                   |
             [VM NIC (ASG: ASG-Web)]
```
