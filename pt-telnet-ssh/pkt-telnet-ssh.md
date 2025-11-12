# 💻 Telnet vs. SSH Configuration Lab (Cisco Packet Tracer)
This project documents a network simulation lab performed in Cisco Packet Tracer to illustrate and compare the configuration, functionality, and security differences between Telnet and SSH for remote device management.
## 🎯Objectives
- Configure basic network connectivity and IP addressing across the topology.
- Set up Telnet access on Router 1 ($\text{R}1$), emphasizing its simplicity and inherent lack of security (clear-text transmission).
- Set up SSH v2 access on Router 2 ($\text{R}2$), highlighting the need for hostname, domain-name, local user database, and cryptographic key generation for secure, encrypted access.
- Implement Static Routing to ensure end-to-end connectivity between different subnets.
- Use Packet Tracer's Simulation Mode to visually verify the clear-text nature of Telnet and the encrypted nature of SSH.
- Perform practical troubleshooting on inter-network communication (ping and connectivity issues).
## 🌐 TopologyThe network consists of two LANs connected by two routers. 
The administrator PC accesses both routers remotely via different protocols.

Device,Interface,IP Address,Subnet Mask,Description
PC_Admin,NIC,192.168.1.10,255.255.255.0,Admin LAN
Router1 (R1),G0/0,192.168.1.1,255.255.255.0,Towards Admin LAN
Router1 (R1),G0/1,10.0.0.1,255.255.255.0,Towards R2
Router2 (R2),G0/0,10.0.0.2,255.255.255.0,Towards R1
Router2 (R2),G0/1,192.168.2.1,255.255.255.0,Towards Remote LAN
PC_Remote,NIC,192.168.2.10,255.255.255.0,Remote LAN

## ⚙️ Key ConfigurationsProtocol
Protocol,Device,Key Commands
Telnet,R1,line vty 0 4 → password cisco → login → transport input telnet
SSH,R2,hostname R2 → ip domain-name lab.local → username admin secret Admin123 → crypto key generate rsa → ip ssh version 2 → line vty 0 4 → login local → transport input ssh
Routing,R1,ip route 192.168.2.0 255.255.255.0 10.0.0.2
Routing,R2,ip route 192.168.1.0 255.255.255.0 10.0.0.1
Priv. Mode,R1,enable password class
Priv. Mode,R2,enable secret root123
## ✅ Testing
Device,Command,Expected Result,Protocol,Security
PC_Admin,telnet 192.168.1.1,Successful login with password cisco,Telnet,Insecure (Clear-text)
PC_Admin,ssh -l admin 10.0.0.2,Successful login with user admin and password Admin123,SSH,Secure (Encrypted)