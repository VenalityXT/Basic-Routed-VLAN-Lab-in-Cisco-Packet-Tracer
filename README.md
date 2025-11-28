# Network Segmentation & VLAN Configuration  
### Router-on-a-Stick • DHCP • VLAN Trunking • Inter-VLAN Routing  
**Cisco Packet Tracer Technical Lab Guide & Professional Documentation**

[![Cisco IOS](https://img.shields.io/badge/Platform-Cisco_IOS-1ba0d7?logo=cisco)](#)
[![Packet Tracer](https://img.shields.io/badge/Tool-Packet_Tracer-0a7cff?logo=cisco)](#)
[![Routing & Switching](https://img.shields.io/badge/Focus-Routing_&_Switching-green)](#)
[![DHCP](https://img.shields.io/badge/Service-DHCP_Automation-yellow)](#)
[![VLANs](https://img.shields.io/badge/Feature-VLAN_Segmentation-orange)](#)

---

# Project Overview

This project demonstrates how to build a segmented small-business network in Cisco Packet Tracer using **VLANs**, **DHCP**, **router-on-a-stick**, and **802.1Q trunking**. The design isolates client devices into separate broadcast domains while still allowing communication across VLANs via a router.

The router provides:

- DHCP services  
- Subinterfaces for inter-VLAN routing  
- Gateway addresses for VLAN 10 and VLAN 20  

The switch provides:

- VLAN segmentation  
- Access ports  
- A trunk to the router  

The PC receives its network configuration dynamically from the router based on its VLAN membership. Each step in this document includes **deep explanations** of commands and concepts to reinforce practical understanding.

<img width="1920" height="1032" alt="PacketTracer_Topology_1" src="https://github.com/user-attachments/assets/c8618d25-8a99-405b-bc02-3d5d42435baa" />

---

# 1. Logical Topology Description

The physical connections in this lab consist of:

- **Router0 GigabitEthernet0/0/0 ↔ Switch0 FastEthernet0/1** (trunk link)  
- **Switch0 FastEthernet0/2 ↔ PC0** (access link in VLAN 10)  

This topology mirrors a traditional single-switch, single-router environment in which the router is responsible for routing between VLANs because the switch operates at Layer 2.

---

# 2. Router Initialization & Physical Interface Setup

When the router boots, IOS presents the System Configuration Dialog. We bypass it to configure the device manually from privileged EXEC mode.

<img width="702" height="712" alt="Step 1" src="https://github.com/user-attachments/assets/66b66fbd-ac6a-418d-8f19-786a17a097c3" />

### Commands Used

```bash
enable
configure terminal
interface gigabitEthernet0/0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

### Explanation of Each Command

**`enable`**  
Moves the session from *user EXEC* (`Router>`) to *privileged EXEC* (`Router#`).  
Privileged EXEC mode allows access to configuration commands, routing tables, debugging, and system controls.

**`configure terminal`**  
Enters *global configuration mode*, where permanent system and interface settings are applied.

**`interface gigabitEthernet0/0/0`**  
Selects the physical port used to connect the router to the switch.  
Cisco interface naming convention:  
- **gigabitEthernet** → type  
- **0/0/0** → slot / subslot / port  

**`ip address 192.168.1.1 255.255.255.0`**  
Temporarily assigns an address to bring the interface up for initial testing.  
This is not the final VLAN gateway address.

**`no shutdown`**  
Activates the interface. Router interfaces default to *administratively down*.

### Engineering Note  
The address `192.168.1.1/24` is not used later because VLAN 10 requires a gateway in the **192.168.10.0/24** network.

---

# 3. DHCP Configuration for VLAN 10

The router distributes IP addresses automatically to VLAN 10 devices.

<img width="702" height="712" alt="Step 2" src="https://github.com/user-attachments/assets/9ef2459e-5303-4ae5-9b72-984154bf6d55" />

### Commands Used

```bash
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

### Explanation of Each Command

**`ip dhcp excluded-address`**  
Prevents DHCP from assigning important addresses.  
`.1` is the gateway; `.10` reserved for static use.

**`ip dhcp pool VLAN10_Pool`**  
Creates a named pool for VLAN 10.  
Naming the pool after the VLAN is a best practice for clarity.

**`network <network> <mask>`**  
Defines the IP subnet DHCP can allocate from.

**`default-router`**  
The gateway that DHCP clients use (router subinterface for VLAN 10).

**`dns-server 8.8.8.8`**  
Provides external name resolution via Google DNS.

### Simple Troubleshooting Tip 💡  
If a VLAN10 client does not receive an address, ensure the PC is set to **DHCP**, not static IP mode.

---

# 4. VLAN Creation & Access Port Assignment on the Switch

We created VLAN 10 (Marketing) and VLAN 20 (HR), then assigned the PC’s port (Fa0/2) to VLAN 10.

<img width="702" height="712" alt="Step 3" src="https://github.com/user-attachments/assets/f89a1423-c60b-4366-a767-fc10f2270428" />

### Commands Used

```bash
vlan 10
 name Marketing

vlan 20
 name HR

interface fa0/2
 switchport mode access
 switchport access vlan 10
 no shutdown
```

### Explanation of Each Command

**`vlan 10` / `vlan 20`**  
Creates VLAN entries in the switch’s VLAN database.

**`switchport mode access`**  
Configures the port as an access port—untagged traffic only.

**`switchport access vlan 10`**  
Places the PC into VLAN 10’s broadcast domain.

**`no shutdown`**  
Enables the port at Layer 1 and Layer 2.

### Engineering Note  
An access port must belong to exactly one VLAN.

---

# 5. Layer 2 Verification (VLANs and Trunks)

We verified VLAN assignments and trunk readiness.

<img width="702" height="712" alt="Step 5 Ping VLANs" src="https://github.com/user-attachments/assets/2f66d848-a0b1-4561-8f91-06cbd673933f" />

### Commands Used

```bash
show vlan brief
show interfaces trunk
show ip interface brief
```

### What These Commands Reveal

- VLAN 10 exists and includes Fa0/2  
- Fa0/1 is operational and capable of trunking  
- All necessary interfaces are up  

### Simple Troubleshooting Tip 💡  
If `Fa0/2` appears in VLAN 1, the PC will not receive a VLAN10 DHCP address.

---

# 6. Router-on-a-Stick Subinterface Configuration

We created subinterfaces to route between VLANs using 802.1Q tagging.

<img width="702" height="712" alt="Step 1" src="https://github.com/user-attachments/assets/0051c284-ebfd-4e05-a8b4-42da4e74e6af" />

### Commands Used

```bash
interface gigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface gigabitEthernet0/0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```

### Explanation of Each Command

**`interface g0/0/0.10`**  
Creates a logical interface for VLAN 10 on the physical port.

**`encapsulation dot1Q 10`**  
Tells the router to expect VLAN 10–tagged frames.

**`ip address`**  
Assigns the gateway address for that VLAN.

This establishes:

- VLAN10 gateway = `192.168.10.1`  
- VLAN20 gateway = `192.168.20.1`  

### Engineering Note  
Each VLAN requires a unique Layer 3 gateway in its own subnet for routing to function.

---

# 7. Configuring the Trunk Link Between Router and Switch

We convert Fa0/1 into a trunk to carry VLAN tags.

<img width="702" height="701" alt="Step 5 vlan brief interfaces trunk Switch" src="https://github.com/user-attachments/assets/dfdb43a9-9772-4ccd-92b4-8f9fb69ea18a" />

### Commands Used

```bash
interface fa0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown
```

### Explanation of Each Command

**`switchport mode trunk`**  
Enables 802.1Q trunking on the interface.

**`allowed vlan all`**  
Permits VLAN10 and VLAN20 to traverse the trunk.

**`no shutdown`**  
Ensures the trunk is active.

### Simple Troubleshooting Tip 💡  
If routing fails, ensure Fa0/1 is not accidentally configured as an access port.

---

# 8. PC DHCP Configuration

<img width="702" height="712" alt="Step 4" src="https://github.com/user-attachments/assets/463d9cf1-bd2f-4ee4-aa16-e337aa7c4e6c" />

After setting the PC to obtain an IP via DHCP, it successfully received:

- IP: `192.168.10.2`  
- Mask: `255.255.255.0`  
- Gateway: `192.168.10.1`  
- DNS: `8.8.8.8`  

This confirms DHCP pool accuracy and correct VLAN membership.

---

# 9. Final Connectivity Validation

We validated end-to-end routing and DHCP operation.

![Ping Tests]("images/Step 5 Ping VLANs.png")

### Successful Tests

- `ping 192.168.10.1` → reached VLAN10’s gateway  
- `ping 192.168.20.1` → confirmed inter-VLAN routing via the router
  
<img width="702" height="156" alt="Step 5 dhcp binding Router" src="https://github.com/user-attachments/assets/330f9ad7-d4ae-49db-8910-9c790866d616" />

`show ip dhcp binding` confirmed the DHCP lease for `192.168.10.2`.

---

# 10. Bonus Expansion: Multi-Network Communication via ISP Router & Static Routing

In this extended portion of the original project, we expanded the lab to simulate **two separate local networks** communicating across an **intermediate ISP router**, modeling how real organizations interconnect geographically separated sites. This enhancement demonstrates routed WAN links, IP addressing schemes for point-to-point connections, and static routing tables that enable full end-to-end communication.

This section follows the same structure and professional formatting style as the original VLAN and router-on-a-stick lab.

---

## 10.1 Objective of the Expansion

The purpose of this expansion was to build on the original segmented network by introducing:

- A multi-router environment  
- An ISP transit network  
- `/30` WAN subnets for point-to-point routing  
- Static routes to enable cross-LAN communication  
- Simulation mode validation using ARP + ICMP filters only  

By completing this extension, we created a small-scale representation of how two remote LANs communicate through a service provider using IPv4 routing tables.

---

## 10.2 Expanded Logical Topology

The final design consists of **three routers**, **two switches**, and **two workstations**, arranged into two independent LANs joined by an ISP core.

<img width="1920" height="1032" alt="S1" src="https://github.com/user-attachments/assets/c696c643-3bfd-471e-be82-39ef18705366" />

## Left-Side LAN (Network A)
- **RT-01** – Edge router for Network A  
- **SW-01** – Access switch  
- **PC-01** – Workstation  
- **LAN Subnet:** `192.168.100.0/24`  
- **Gateway:** `192.168.100.1`  
- **Device Abbreviations:**  
  - **RT** = Router  
  - **SW** = Switch  
  - **PC** = Personal Computer  

## Right-Side LAN (Network B)
- **RT-02** – Edge router for Network B  
- **SW-02** – Access switch  
- **PC-02** – Workstation  
- **LAN Subnet:** `192.168.200.0/24`  
- **Gateway:** `192.168.200.1`  

## ISP Transit Router
- Provides interconnection between both networks  
- No DHCP, VLANs, or switching  
- Pure Layer 3 forwarding  
- Uses `/30` networks for efficient point-to-point addressing  

---

## 10.3 WAN Point-to-Point Subnetting

We assigned `/30` subnets to each router-to-router link. A `/30` provides exactly:

- 1 usable IP per device (2 total hosts)  
- 1 network ID  
- 1 broadcast address  

This is the industry standard for WAN serial or Ethernet point-to-point connections.

### Subnet Allocation
- **RT-01 ↔ ISP:** `10.0.0.0/30`  
  - RT-01 = `10.0.0.1`  
  - ISP  = `10.0.0.2`  

- **RT-02 ↔ ISP:** `10.0.0.4/30`  
  - ISP   = `10.0.0.5`  
  - RT-02 = `10.0.0.6`  

These subnets form the WAN backbone that carries cross-LAN traffic.

---

## 10.4 Static Routing Table Configuration

Static routes were created on each router to inform them of all remote networks.

## RT-01 Routing Table
Routes traffic for Network B and next-hop WAN:
```bash
ip route 192.168.200.0 255.255.255.0 10.0.0.2
```

<img width="702" height="712" alt="image" src="https://github.com/user-attachments/assets/74e8de0e-7408-44be-974f-c05be6d50684" />

## RT-02 Routing Table
Routes traffic for Network A and next-hop WAN:
```bash
ip route 192.168.100.0 255.255.255.0 10.0.0.5
```

<img width="702" height="712" alt="image" src="https://github.com/user-attachments/assets/c992bc5f-83fb-48cc-ac33-f267c129c119" />

## ISP Routing Table
Directs traffic between both remote LANs:
```bash
ip route 192.168.100.0 255.255.255.0 10.0.0.1
ip route 192.168.200.0 255.255.255.0 10.0.0.6
```

<img width="702" height="712" alt="image" src="https://github.com/user-attachments/assets/c6c29fc3-f2d2-4518-b022-e7f9424e947d" />

This created full bidirectional reachability.

---

## 10.5 DHCP & LAN Connectivity Verification

Both LANs obtained correct DHCP leases from their respective routers:

### PC-01 (Network A)
- IP: `192.168.100.X`  
- Mask: `255.255.255.0`  
- Gateway: `192.168.100.1`  

<img width="702" height="712" alt="image" src="https://github.com/user-attachments/assets/4f900ebb-6b5a-470a-83f3-1d061793c3fa" />

### PC-02 (Network B)
- IP: `192.168.200.X`  
- Mask: `255.255.255.0`  
- Gateway: `192.168.200.1`  

<img width="702" height="712" alt="image" src="https://github.com/user-attachments/assets/3fd41e8a-ed3b-480c-99d9-0900392b1e07" />

Correct DHCP operation confirmed that the VLAN-based LAN configuration from the original project transferred cleanly into this multi-router environment.

---

## 10.6 End-to-End Connectivity Testing

After routing tables were configured, both PCs were able to:

- Ping each other  
- Ping the opposite router  
- Ping across the ISP  
- Perform ARP resolution correctly  

Simulation mode was used for validation with only:

- **ARP**  
- **ICMP**

enabled in the Event Filter, allowing a clean visual workflow.

### Observed Packet Flow
1. PC-01 sends ARP request  
2. Local switch forwards frame  
3. RT-01 resolves MAC  
4. Packet travels to ISP router  
5. ISP forwards to RT-02  
6. RT-02 performs ARP for PC-02  
7. ICMP Echo Reply travels back the same path  

This confirmed flawless multi-router end-to-end communication.

---

## 10.7 Bonus Video Demonstration Placeholder

Below is a placeholder section for attaching your final Packet Tracer simulation video:

**▶ WAN Simulation Video Demonstration**  
*This video demonstrates successful ARP + ICMP flow in Simulation Mode with only ARP and ICMP enabled in the event filter, showing full packet traversal between two independent networks across an ISP transit router.*

https://github.com/user-attachments/assets/4df6dc48-300d-4fcc-9fd7-f9f1e2eb3059

---

## 10.8 Summary of the Multi-Network Expansion

This bonus section built on the original VLAN and router-on-a-stick project by introducing:

- A three-router, two-LAN architecture  
- Realistic ISP transit routing  
- `/30` point-to-point subnets  
- Static routing tables enabling remote network communication  
- Successful PC-to-PC communication across separate networks  
- Simulation verification using ARP + ICMP filters  

This addition transforms the project from a single-router VLAN environment into a full **multi-router enterprise WAN simulation**, demonstrating core skills used in routing, addressing, WAN design, and troubleshooting.

---

# Deep Command Reference

### EXEC & Global Modes
- `enable` — privileged mode  
- `configure terminal` — global configuration mode  

### Router Interfaces
- `interface g0/0/0` — physical router port  
- `interface g0/0/0.X` — VLAN subinterface  
- `encapsulation dot1Q X` — VLAN tagging  
- `ip address` — assigns Layer 3 gateway  
- `no shutdown` — activates interface  

### DHCP
- `ip dhcp excluded-address` — reserves addresses  
- `ip dhcp pool NAME` — creates DHCP pool  
- `network` — defines DHCP subnet  
- `default-router` — client gateway  
- `dns-server` — DNS distribution  

### Switch VLANs
- `vlan X` — creates VLAN  
- `switchport mode access` — access port  
- `switchport access vlan X` — VLAN assignment  

### Trunk Link
- `switchport mode trunk` — 802.1Q trunk  
- `switchport trunk allowed vlan all` — allow VLANs across the trunk  

### Verification
- `show vlan brief`  
- `show interfaces trunk`  
- `show ip interface brief`  
- `show ip dhcp binding`  

---

# Summary

This lab deployed a complete VLAN-segmented network using router-on-a-stick routing, DHCP automation, and trunking. By configuring VLANs on the switch, establishing subinterfaces with 802.1Q encapsulation on the router, and enabling DHCP pools mapped to each VLAN, we achieved a fully functional multi-VLAN environment. The network supports dynamic addressing, isolation between broadcast domains, and inter-VLAN communication through a centralized router. Each command was explained both in context and in a consolidated reference to ensure clarity and reinforce enterprise-ready networking concepts.

