# Enterprise Network Segmentation Lab

## Overview
This lab simulates a small enterprise network in Cisco Packet Tracer to practice and document core networking concepts such as VLAN segmentation, secure trunking, inter-VLAN routing on a Layer 3 switch, and traffic isolation using ACLs.

The goal is not only to configure the network, but also to understand the reasoning behind each design choice and the troubleshooting mindset required in real-world administration.

## Table of Contents
- [Overview](#overview)
- [Topology](#topology)
- [VLAN and IP Addressing Plan](#vlan-and-ip-addressing-plan)
- [Port Mapping](#port-mapping)
- [Configuration Summary](#configuration-summary)
- [Key Concepts](#key-concepts)
- [Guest VLAN Isolation with ACL](#guest-vlan-isolation-with-acl)
- [Verification and Testing](#verification-and-testing)
- [Roadmap](#roadmap)
- [Tech Stack](#tech-stack)

## Topology

```text
Router ISP (optional)
        |
Layer 3 Switch (Cisco 3560)
   /                 \
  /                   \
Access Switch 1      Access Switch 2
(Cisco 2960)         (Cisco 2960)
|   |   |             |   |   |
Admin Admin Technician Technician Guest Guest
                        |
                    Server (VLAN 40)
```

### Devices
- Cisco 3560: Layer 3 switch for inter-VLAN routing, SVIs, and ACLs
- Cisco 2960 x2: Layer 2 access switches for end-device connectivity
- 6 PCs + 1 Server: Devices distributed across four departments
- Router ISP: Optional and not configured in this iteration

## VLAN and IP Addressing Plan

| VLAN | Department | Network | Gateway (SVI) |
| --- | --- | --- | --- |
| 10 | Administration | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Technicians | 192.168.20.0/24 | 192.168.20.1 |
| 30 | Guest | 192.168.30.0/24 | 192.168.30.1 |
| 40 | Servers | 192.168.40.0/24 | 192.168.40.1 |

> Every department is its own broadcast domain. Communication between VLANs occurs only through the Layer 3 switch SVIs, never directly at Layer 2.

## Port Mapping

### Access Switch 1 (2960)

| Port | Device | VLAN |
| --- | --- | --- |
| Fa0/1 | PC A – Admin | 10 |
| Fa0/2 | PC B – Admin | 10 |
| Fa0/3 | PC C – Technician | 20 |
| Gi0/1 | Trunk to 3560 | — |

### Access Switch 2 (2960)

| Port | Device | VLAN |
| --- | --- | --- |
| Fa0/1 | PC D – Technician | 20 |
| Fa0/2 | PC E – Guest | 30 |
| Fa0/3 | PC F – Guest | 30 |
| Gi0/2 | Trunk to 3560 | — |

### Layer 3 Switch (3560)

| Port | Connection | Mode |
| --- | --- | --- |
| Gi0/1 | Access Switch 1 | Trunk |
| Gi0/2 | Access Switch 2 | Trunk |
| Fa0/1 | Server (VLAN 40) | Access |

> VLANs are assigned per port, not per switch. The same physical 2960 switch can host ports in different VLANs.

## Configuration Summary

### 1. Create VLANs

```cisco
vlan 10
 name ADMIN
vlan 20
 name TECHNICIANS
vlan 30
 name GUEST
vlan 40
 name SERVERS
```

Each switch maintains its own local VLAN database. If an undefined VLAN is assigned to an access port, IOS may auto-create it with a default name such as VLAN00XX.

### 2. Configure secure trunk links

```cisco
! On the 3560
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
```

```cisco
! On each 2960
interface GigabitEthernet0/1
 switchport mode trunk
 switchport nonegotiate
```

`switchport nonegotiate` disables DTP and helps prevent trunk negotiation abuse such as switch spoofing.

### 3. Configure access ports

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
```

### 4. Configure SVIs for inter-VLAN routing

```cisco
ip routing

interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

Repeat the same pattern for VLANs 20, 30, and 40.

### 5. Apply an ACL to isolate the Guest VLAN

```cisco
ip access-list extended GUEST_BLOCK
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip any any

interface vlan 30
 ip access-group GUEST_BLOCK in
```

## Key Concepts

### VLANs and Broadcast Domains
- A VLAN is a separate broadcast domain.
- Inter-VLAN communication requires a Layer 3 device such as a multilayer switch.
- Each switch has its own VLAN database; creating a VLAN on one switch does not automatically propagate it to its neighbors.

### Trunking
- Legacy switches such as the 3560 support both ISL and 802.1Q, so `switchport trunk encapsulation dot1q` is required.
- Newer switches such as the 2960 support 802.1Q only.
- DTP can auto-negotiate trunk status; disabling it with `switchport nonegotiate` is recommended on intentional trunk links.

### STP
- STP operates at Layer 2 and helps prevent loops by blocking redundant paths.
- A blocked port is an availability issue, not a security issue.

### SVIs and Routing
- An SVI comes up only when the VLAN exists, the port is active, and the trunk carries the VLAN.
- `ip routing` must be enabled on the Layer 3 switch for inter-VLAN routing to work.
- Always check `show ip route` before testing connectivity.

### ARP and Packet Flow
- The first packet in a ping is typically an ARP request used to resolve the next-hop MAC address.
- For inter-VLAN traffic, the source host ARPs for its default gateway, not the remote host.
- The IP header remains unchanged end-to-end, while the Ethernet frame is rebuilt at each Layer 3 hop.

### Security Design
- Routing is not the same as security.
- ACL order matters because IOS evaluates rules top-down and stops at the first match.
- Extended ACLs end with an implicit deny; an explicit permit is often needed at the end.
- Applying ACLs as close as possible to the source is more efficient.

## Guest VLAN Isolation with ACL

### Objective
Guest users in VLAN 30 should be able to reach the rest of the network, but must never reach the Servers VLAN (VLAN 40).

### Test Results

| Test | Expected Result |
| --- | --- |
| Guest host → Server | Blocked |
| Guest host → Technician host | Allowed |

## Verification and Testing

| Command | Purpose |
| --- | --- |
| `show vlan brief` | View VLAN database and access-port assignments |
| `show interfaces trunk` | Confirm trunk status, encapsulation, and allowed VLANs |
| `show interfaces <if> switchport` | Inspect per-port switchport behavior |
| `show ip interface brief` | Verify interface and SVI status |
| `show ip route` | Confirm the routing table |
| `show access-lists` | Review ACL contents and hit counters |
| `ping` | Validate end-to-end connectivity |

## Roadmap
This lab is intentionally scoped to VLANs, trunking, inter-VLAN routing, and ACL security. The following topics are left for a future, troubleshooting-focused lab:

- Collision domains
- `show mac address-table`
- `show arp` on the Layer 3 switch
- A deliberate break-and-repair troubleshooting exercise
- Native VLAN consistency checks across switches
- Optional ISP router integration with default routing and NAT
- DHCP deployment for end-user devices

## Tech Stack
- Cisco IOS
- Cisco Packet Tracer
- VLANs and 802.1Q
- Inter-VLAN routing
- Extended ACLs
