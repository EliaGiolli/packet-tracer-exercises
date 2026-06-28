# Network Infrastructure: HQ to Remote IoT/Worker Integration

## 🎯 Project Overview
This project simulates a hybrid enterprise network designed to connect a central headquarters (HQ) with a remote/home office environment. The objective is to provide secure access to corporate resources through a VPN-like tunnel while isolating domestic IoT devices for security reasons.

## 📸 Network Topology
The network topology is implemented in Cisco Packet Tracer. Make sure the filename matches exactly.

## 🛠️ Design Goals
- **Segmentation**: Isolate IoT traffic within the home network.
- **VPN-like Access**: Enable bidirectional access only for the remote worker's laptop.
- **Security**: Implement NAT and static routing to simulate a realistic production environment.

## 🔍 Troubleshooting Log
During implementation, several technical issues were identified and resolved. The main hardening points are summarized below:

| Category | Problem encountered | Solution adopted |
| --- | --- | --- |
| Physical | Interface mismatches | Verified through `show ip int brief` and replaced the faulty hardware |
| Routing | "Invalid next hop" on the wireless router | Replaced the consumer-grade router with a Cisco 2911/1941 router |
| NAT | Packets were not being translated | Corrected the `ip nat inside/outside` configuration |
| Connectivity | ICMP timeout | Reviewed ACLs and verified the default gateway |

## ✅ Key Issues Resolved
- **Wireless Router Illusion**: The Packet Tracer wireless router is a consumer-grade appliance without a routing table. The solution involved using a real Cisco router paired with an access point.
- **IoT Logic**: IoT devices in Packet Tracer do not display physical links. Connectivity was validated at the logical level through DHCP and SSID behavior.
- **VPN Simulation**: Static cross-routing was implemented to enable traffic between the 192.168.50.0/24 (Home) and 10.10.10.0/24 (HQ) subnets.

## 🚀 Future Roadmap
- Implement a real IPSec tunnel.
- Configure zone-based firewall (ZBF) policies at the HQ site.
- Automate device configuration using Python and Netmiko.

This project was created for educational purposes and infrastructure simulation.

