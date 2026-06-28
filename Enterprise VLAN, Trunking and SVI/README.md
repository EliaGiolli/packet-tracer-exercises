🌐 Enterprise Network Segmentation Lab — VLANs, Inter-VLAN Routing & ACL Security

Show Image
Show Image
Show Image
Show Image

A small simulated enterprise network built in Cisco Packet Tracer, designed to reproduce — and reason through — the real-world responsibilities of a Network Administrator / Sysadmin: VLAN segmentation, secure trunking, inter-VLAN routing on a Layer 3 switch, and traffic isolation via ACLs.

This README is not just a config dump — it's a checklist of the concepts and habits a network engineer should always have in mind, distilled from building this lab end-to-end.


📋 Table of Contents


🏗️ Topology
📐 VLAN & IP Addressing Plan
🔌 Port Mapping
⚙️ Configuration Summary
🧠 Concepts Every Network Admin Must Keep in Mind
🔐 Security: Guest VLAN Isolation (ACL)
✅ Verification & Testing
🗺️ Roadmap / Future Work



🏗️ Topology

                     Router ISP (optional)
                            |
                     Layer 3 Switch (Cisco 3560)
                     /                   \
             trunk /                     \ trunk
                  /                       \
         Access Switch 1            Access Switch 2
           (Cisco 2960)              (Cisco 2960)
          |      |      |            |      |      |
       Admin   Admin Technician   Technician Guest Guest
                              |
                          Server (VLAN 40, direct link to L3 switch)

DeviceRoleCisco 3560Layer 3 Switch — inter-VLAN routing, SVIs, ACLsCisco 2960 × 2Layer 2 Access Switches — end-device connectivity6× PC + 1 ServerEnd devices across 4 departmentsRouter ISPOptional, not configured in this iteration


📐 VLAN & IP Addressing Plan

VLANDepartmentNetworkGateway (SVI)10🧑‍💼 Administration192.168.10.0/24192.168.10.120🛠️ Technicians192.168.20.0/24192.168.20.130🌐 Guest192.168.30.0/24192.168.30.140🖥️ Servers192.168.40.0/24192.168.40.1


💡 Every department is its own broadcast domain. Communication between VLANs only happens through the Layer 3 switch's SVIs — never directly at Layer 2.




🔌 Port Mapping

Access Switch 1 (2960)

PortDeviceVLANFa0/1PC A – Admin10Fa0/2PC B – Admin10Fa0/3PC C – Technician20Gig0/1Trunk → 3560—

Access Switch 2 (2960)

PortDeviceVLANFa0/1PC D – Technician20Fa0/2PC E – Guest30Fa0/3PC F – Guest30Gig0/2Trunk → 3560—

Layer 3 Switch (3560)

PortConnectionModeGig0/1Trunk → Access Switch 1trunkGig0/2Trunk → Access Switch 2trunkFa0/1Server (VLAN 40)access


⚠️ VLANs are assigned per port, never per switch. The same physical 2960 can — and does — host ports in different VLANs.




⚙️ Configuration Summary

<details>
<summary><b>1. Create VLANs in the database</b></summary>
vlan 10
 name ADMIN
vlan 20
 name TECHNICIANS
vlan 30
 name GUEST
vlan 40
 name SERVERS


Each switch keeps its own local, independent VLAN database. If you assign an undefined VLAN to an access port, IOS auto-creates it with a default name (VLAN00XX).



</details>
<details>
<summary><b>2. Secure trunk links (3560 ↔ 2960)</b></summary>
! On the 3560 (supports both ISL and 802.1Q → encapsulation must be set explicitly)
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate

! On each 2960 (802.1Q only → no encapsulation command, no such option exists)
interface GigabitEthernet0/1
 switchport mode trunk
 switchport nonegotiate


nonegotiate disables DTP (Dynamic Trunking Protocol) — preventing a rogue device from negotiating a trunk and pulling off VLAN hopping via switch spoofing.



</details>
<details>
<summary><b>3. Access ports (toward PCs / Server)</b></summary>
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10

</details>
<details>
<summary><b>4. SVIs — Layer 3 gateways</b></summary>
ip routing

interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
! repeat for vlan 20 / 30 / 40


IP addresses never go on physical ports — only on logical VLAN interfaces (SVIs), which act as the gateway of each department.



</details>
<details>
<summary><b>5. ACL — isolate Guest from Servers</b></summary>
ip access-list extended GUEST_BLOCK
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip any any

interface vlan 30
 ip access-group GUEST_BLOCK in

</details>

🧠 Concepts Every Network Admin Must Keep in Mind

A working network is not enough — understanding why it works (or breaks) is what separates a script-runner from a sysadmin. Key takeaways from this lab:

VLANs & Broadcast Domains


A VLAN is a separate broadcast domain. Without VLANs, every host sees every ARP broadcast on the wire.
Inter-VLAN communication is never possible at Layer 2 — it always requires a Layer 3 device.
Each switch has its own independent VLAN database — creating a VLAN on one switch does not propagate it to neighbors.


Trunking


🔑 Encapsulation quirk: legacy switches (e.g. 3560) support both ISL and 802.1Q and require switchport trunk encapsulation dot1q before switchport mode trunk — otherwise IOS rejects the command. Newer switches (2960) support 802.1Q only and don't even have that command.
A trunk forwards every allowed VLAN by default (1–1005), regardless of whether the other side actually has ports in that VLAN. Broadcasts from a VLAN with no local ports on a switch simply get flooded nowhere and die there — by design, not by error.
DTP (Dynamic Trunking Protocol) can auto-negotiate trunk status. Always disable it with switchport nonegotiate on intentional trunk links — leaving negotiation enabled is a textbook VLAN hopping / switch spoofing vector.


Spanning Tree Protocol (STP)


STP operates at Layer 2, not Layer 3 — despite "reasoning" about topology, it works on Ethernet frames (BPDUs), not IP.
STP exists primarily to prevent broadcast storms from physical loops, but it also catches port-type inconsistencies (e.g. one side trunk, the other still access) and blocks the port defensively.
A blocked port is an availability issue, not a security issue — no traffic flows at all, so there's nothing for an attacker to "hook into."


SVIs & Routing


An SVI only comes Up if: the VLAN exists, at least one port in that VLAN is physically active, the trunk actually carries that VLAN, and there are no Layer 2 misconfigurations.
ip routing must be explicitly enabled — without it, even a Layer 3-capable switch behaves like a plain Layer 2 switch.
Always check show ip route before testing connectivity — a missing C (connected) route means a ping is pointless to try.


ARP & Packet Flow


The very first packet of a ping is never ICMP — it's an ARP request resolving the next hop's MAC, not the final destination's.
For inter-VLAN traffic, the source host ARPs for its default gateway, not the remote host — a broadcast ARP never leaves its own VLAN/broadcast domain anyway.
Across a Layer 3 hop: the IP header never changes (source/destination IP stay identical end-to-end); the Ethernet frame is rebuilt at every segment (MAC addresses change at every Layer 3 hop).
TTL decreases by 1 at every Layer 3 hop — a great low-effort way to count hops without traceroute.


Security Design


🚨 Routing ≠ Security. Inter-VLAN routing forwards any traffic between connected networks by default — segmentation alone does not isolate anything. Prove the problem before applying the fix: test the "bad" path before writing the ACL.
ACL ordering matters absolutely. IOS evaluates rules top-down and stops at the first match. A generic permit placed before a specific deny makes the deny dead code — it will never be evaluated.
Every extended ACL ends with an implicit deny ip any any. A list with only one deny line blocks everything else too — you almost always need an explicit permit ip any any at the end.
in / out on an ACL are always relative to the interface it's applied on, not to traffic direction in general. Filtering as close as possible to the traffic source (ingress on the source VLAN's SVI) is more efficient — the packet is dropped before any routing lookup is even performed.
A Destination host unreachable message tells you exactly where the packet was dropped — the sender IP is the interface that discarded it. Read it during troubleshooting, don't ignore it.


Static vs Dynamic Addressing


Servers → always static. ACLs, DNS records, firewall rules, and monitoring all point to a fixed IP — a silent change breaks everything downstream.
End-user devices → often DHCP in production, especially as a network grows past what's manageable with manual IP tracking.



🔐 Security: Guest VLAN Isolation (ACL)

Goal: Guest (VLAN 30) must reach the rest of the network normally, but must never reach the Servers VLAN (VLAN 40).

ip access-list extended GUEST_BLOCK
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip any any
!
interface vlan 30
 ip access-group GUEST_BLOCK in

ConceptExplanationWildcard maskBitwise inverse of the subnet mask (255 − octet). 0.0.0.255 for a /24 = "match the network exactly, any host."Rule orderSpecific deny first, generic permit last — reversed order disables the deny entirely.Applied in on VLAN 30's SVIDrops the packet as close to the source as possible, before any routing decision is made — more efficient than filtering out on VLAN 40.

Test results

TestExpectedResultGuest (.30.10) → Server (.40.10)❌ BlockedDestination host unreachable from 192.168.30.1 — 100% lossGuest (.30.10) → Technician (.20.10)✅ Allowed0% loss (after ARP resolution on the first attempt)


✅ Verification & Testing

CommandPurposeshow vlan briefVLAN database & access port assignments per switchshow interfaces trunkTrunk status, encapsulation, native VLAN, allowed/active VLANsshow interfaces <if> switchportPer-port detail, incl. Negotiation of Trunking: Off to confirm nonegotiateshow ip interface briefUp/Down status & IP of every interface and SVIshow ip routeRouting table — confirms reachability before testing with pingshow access-listsACL contents and match countersping (Simulation Mode in Packet Tracer)Frame-by-frame proof of ARP resolution, MAC rewriting per hop, and ACL drops


🗺️ Roadmap / Future Work

This lab is intentionally scoped to VLANs → Trunking → Inter-VLAN Routing → ACL security — complete and self-contained enough to explain end-to-end (e.g. in an interview). The following topics are deferred to a separate, troubleshooting-focused lab:


 Collision Domains (distinct from broadcast domains, only briefly mentioned here)
 show mac address-table — inspect dynamically learned entries on both access switches
 show arp on the L3 switch — inspect the ARP cache built up during testing
 A dedicated "break it on purpose, diagnose it from scratch" troubleshooting exercise
 Minor cleanup: rename VLAN0020/VLAN0030 on Access Switch 2; verify native VLAN consistency across both 2960s
 Optional ISP Router integration (default route / NAT)
 DHCP for end-user devices, as a more realistic alternative to static addressing



📚 Tech Stack

Cisco IOS · Cisco Packet Tracer · VLAN / 802.1Q · Inter-VLAN Routing · Extended ACLs


Built as a hands-on CCNA/CCNP-prep lab, with emphasis on understanding the "why" behind every configuration line — not just reproducing commands.