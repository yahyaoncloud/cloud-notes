# Azure VNet Peering, Gateway Transit & Azure Virtual WAN

---

## 1. VNet Peering Mechanics

* **Regional VNet Peering:** Connects VNets within the same Azure region via Azure's private backbone network.
* **Global VNet Peering:** Connects VNets across different Azure regions with low-latency private connectivity (traffic never traverses the public Internet).
* **Characteristics:** Low latency, high bandwidth, non-transitive by default.

### Gateway Transit & Remote Gateways:
* **Allow Gateway Transit:** Configured on the Hub VNet to share its VPN/ExpressRoute Gateway with Peered Spoke VNets.
* **Use Remote Gateways:** Configured on the Spoke VNet to route on-premises traffic through the Hub VNet's VPN Gateway.

---

## 2. Azure Virtual WAN (vWAN) Hub-and-Spoke Topology

Azure Virtual WAN automates large-scale branch-to-Azure, user-to-Azure (P2S VPN), and site-to-site connectivity:
* **Virtual Hub:** Microsoft-managed virtual network resource that terminates VPNs, ExpressRoute, and VNet connections.
* **Azure Firewall Integration:** Deploys managed firewall inside Virtual Hub (Secured Virtual Hub) for centralized routing and egress inspection.
