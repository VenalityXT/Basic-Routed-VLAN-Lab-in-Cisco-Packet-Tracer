# Network Segmentation & VLAN Configuration  
### *Router-on-a-Stick • DHCP • VLAN Trunking • Inter-VLAN Routing*  
**Cisco Packet Tracer Technical Lab Guide + Professional Documentation**

[![Cisco IOS](https://img.shields.io/badge/Platform-Cisco_IOS-1ba0d7?logo=cisco)](#)
[![Packet Tracer](https://img.shields.io/badge/Tool-Packet_Tracer-0a7cff?logo=cisco)](#)
[![Routing & Switching](https://img.shields.io/badge/Focus-Routing_&_Switching-green)](#)
[![DHCP](https://img.shields.io/badge/Service-DHCP_Automation-yellow)](#)
[![VLANs](https://img.shields.io/badge/Feature-VLAN_Segmentation-orange)](#)

---

# Project Overview

This lab demonstrates how to design and configure a segmented network using **Cisco Router-on-a-Stick**, **DHCP automation**, **VLAN segmentation**, and **802.1Q trunking**. The environment mirrors a small business infrastructure where multiple departments operate on separate broadcast domains while sharing the same physical hardware.

In this setup:

- The **router** provides DHCP services and Layer 3 routing.
- The **switch** handles VLAN segmentation and trunking.
- The **PC** dynamically receives IP settings from the router based on its VLAN assignment.

The project includes **deep command explanations** to ensure you understand not only *what* to type, but *why* each step is required.  
This builds a real-world mental model of how VLAN isolation, routing, and DHCP all interact.

![Topology]("images/PacketTracer_Topology_1.png")

---

# 1. Logical Topology

The network consists of:

- **Router0** — Subinterfaces for VLAN10 and VLAN20 + DHCP  
- **Switch0** — VLAN segmentation (10 & 20) + trunk on Fa0/1  
- **PC0** — DHCP client assigned to VLAN10  

This reflects an enterprise-style design where each department is given its own subnet and gateway.

---

# 2. Router Initialization & Interface Activation

When the router boots, IOS asks whether you want to enter the setup dialog. Selecting **“no”** is standard for controlled, manual configuration.

![Step 1]("images/Step 1.png")

### Commands Entered

```bash
enable
configure terminal
interface gigabitEthernet0/0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

### Deep Explanation of What These Commands Do

- **`enable`**  
  - Elevates you from *user EXEC* (`Router>`) to *privileged EXEC* (`Router#`).  
  - Required to view full configurations, run diagnostics, or enter config mode.

- **`configure terminal`** or `conf t` 💡  
  - Enters *global configuration mode*, where you make persistent changes to the device.  
  - From here you can configure interfaces, routing protocols, VLANs, services, etc.

- **`interface gigabitEthernet0/0/0`**  
  - Selects the physical interface that connects to the switch.  
  - Naming convention: `type slot/subslot/port`.

- **`ip address <ip> <mask>`**  
  - Assigns an IP address to the interface.  
  - This creates the router’s identity on that network.

- **`no shutdown`**  
  - Activates the interface — all Cisco router interfaces start *administratively down*.  
  - ⚠️ Without this, the interface will not forward packets even if configured correctly.

### ⚠️ Why 192.168.1.1/24 Will Not Work Later  
Once VLAN10 is created, the PC will be in the **192.168.10.0/24** network.  
A PC cannot communicate with a router using a gateway in a **different** network.  
Thus routing requires a gateway inside its VLAN subnet: **192.168.10.1**.

---

# 3. DHCP Configuration for VLAN10

We configure DHCP to automate client IP assignment.

![Step 2]("images/Step 2.png")

### Commands Entered

```bash
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

### Deep Explanation

- **`ip dhcp excluded-address`**  
  Prevents DHCP from leasing certain addresses.  
  - `.1` = gateway  
  - `.10` = reserved static host

- **`ip dhcp pool VLAN10_Pool`**  
  Creates a pool used exclusively by VLAN10.  
  💡 Pool naming aligned with VLAN ID is an enterprise best practice.

- **`network 192.168.10.0 255.255.255.0`**  
  Specifies the subnet for DHCP leases.

- **`default-router 192.168.10.1`**  
  Supplies the **default gateway** to clients — must match the router’s VLAN10 subinterface.

- **`dns-server 8.8.8.8`**  
  Assigns Google’s DNS to DHCP clients.

---

# 4. VLAN Creation & Switch Access Port Assignment

We now build the segmented Layer 2 domains inside the switch.

![Step 3]("images/Step 3.png")

### Commands Entered

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

### Deep Explanation

- **`vlan 10` / `vlan 20`**  
  Creates two isolated broadcast domains.  
  Devices in VLAN10 cannot communicate with VLAN20 without a router.

- **`switchport mode access`**  
  Tells the switch the port is an endpoint device, not a trunk.

- **`switchport access vlan 10`**  
  Assigns PC0 into the Marketing VLAN.

- **`no shutdown`**  
  Ensures the port is active at Layer 1 & Layer 2.

---

# 5. Layer 2 Verification

![Switch State]("images/Step 5 vlan brief interfaces trunk Switch.png")

### Useful Commands

```bash
show vlan brief
show interfaces trunk
show ip interface brief
```

These confirmed:

- Fa0/2 → correctly assigned to VLAN10  
- Fa0/1 → capable of trunking  
- All switch ports active and operational  

---

# 6. Initial Connectivity Failure

![Ping Failure]("images/Step 5 Ping VLANs.png")

### Why the Ping Failed  
- Router was using **192.168.1.1**  
- PC was using **192.168.10.x**  
- They were in different IP networks  
- No routing occurred between them  

The correct fix: create router subinterfaces.

---

# 7. Router-on-a-Stick Subinterface Configuration

![Subinterfaces]("images/Step 1.png")

### Commands Entered

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

### Deep Explanation

- **`interface g0/0/0.10`**  
  Creates a VLAN10 *logical* interface — VLAN10 traffic is routed here.

- **`encapsulation dot1Q 10`**  
  Enables 802.1Q tagging for VLAN10.  
  💡 Switch marks VLAN membership by tagging frames; router reads those tags.

- **`ip address`**  
  This becomes the **default gateway** for all hosts in that VLAN.

Each VLAN gets its own gateway:  
- VLAN10 → 192.168.10.1  
- VLAN20 → 192.168.20.1  

This is essential because routers forward packets **between** networks.

---

# 8. Configuring the Trunk Link (Switch → Router)

![Trunk]("images/Step 5 vlan brief interfaces trunk Switch.png")

### Commands Entered

```bash
interface fa0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown
```

### Deep Explanation

- **`switchport mode trunk`**  
  Makes Fa0/1 a trunk port capable of carrying multiple VLANs.

- **`allowed vlan all`**  
  Permits all VLANs (including 10 and 20) to reach the router.

💡 Without trunking, router-on-a-stick cannot function.

---

# 9. PC DHCP Configuration

![PC DHCP]("images/Step 4.png")

Once the PC NIC is set to DHCP, it receives:

- **IP:** 192.168.10.2  
- **Mask:** 255.255.255.0  
- **Gateway:** 192.168.10.1  
- **DNS:** 8.8.8.8  

All supplied by the `VLAN10_Pool` DHCP configuration.

---

# 10. Successful Connectivity Tests

![Successful Ping]("images/Step 5 Ping VLANs.png")

We validated routing by pinging both VLAN gateway interfaces:

- 192.168.10.1 → local VLAN default gateway  
- 192.168.20.1 → remote VLAN gateway (inter-VLAN routing)  

![Bindings]("images/Step 5 dhcp binding Router.png")

`show ip dhcp binding` confirmed the DHCP lease for 192.168.10.2.

---

# Deep Command Reference (Consolidated)

### EXEC & Configuration Modes  
- **enable** — privileged EXEC mode  
- **configure terminal** — global config mode  

### Router Interfaces  
- **interface g0/0/0** — physical port to switch  
- **interface g0/0/0.10** — VLAN10 subinterface  
- **encapsulation dot1Q X** — enables tagging  

### IP Addressing  
- **ip address X Y** — assigns Layer 3 identity  
- **no shutdown** — activates interface  

### DHCP  
- **ip dhcp excluded-address** — reserve IPs  
- **ip dhcp pool NAME** — create VLAN-specific pool  
- **default-router** — client gateway  
- **dns-server** — DNS for clients  

### Switch VLANs  
- **vlan X** — create VLAN  
- **switchport mode access** — endpoint port  
- **switchport access vlan X** — assign VLAN  

### Trunking  
- **switchport mode trunk** — allows VLAN tagging  
- **switchport trunk allowed vlan all** — permits all VLANs  

### Verification  
- **show vlan brief** — VLAN membership  
- **show interfaces trunk** — trunk status  
- **show ip dhcp binding** — DHCP leases  

---

# 🏁 Final Summary

This project implemented a fully segmented network using VLANs, DHCP, and router-on-a-stick routing. By configuring subinterfaces, trunk ports, DHCP pools, and access ports, we created a scalable multi-VLAN environment identical to what is used in enterprise networks. The detailed command explanations illuminate how Layer 2 segmentation, Layer 3 routing, and DHCP automation work together to deliver a clean, efficient, and secure network design.

