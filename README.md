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

This project demonstrates how to design and configure a segmented small-enterprise network using **VLANs**, **DHCP**, **router-on-a-stick**, and **802.1Q trunking**. The goal is to create isolated broadcast domains on a Layer 2 switch and provide routing and automated IP assignment through a Cisco router.

The lab consists of:

- A router performing Layer 3 routing and DHCP services  
- A switch performing VLAN segmentation and trunking  
- A PC in VLAN 10 receiving its network configuration dynamically  

This document walks through each configuration step, explains every command, and includes engineering notes to provide context and reinforce best practices. Troubleshooting notes are included only for simple, realistic mistakes engineers commonly encounter.

![Topology]("images/PacketTracer_Topology_1.png")

---

# 1. Logical Topology Description

The network topology includes:

- **Router0** connected to the switch via GigabitEthernet0/0/0  
- **Switch0** interconnected using FastEthernet ports  
- **PC0** connected on Fa0/2 and assigned to VLAN 10  

The design reflects a scalable architecture where additional VLANs (like HR on VLAN 20) can easily be added later.

---

# 2. Router Initialization & Physical Interface Setup

The router boots into IOS and prompts for the System Configuration Dialog. We manually configured the device through privileged EXEC and global configuration modes.

![Step 1]("images/Step 1.png")

### Commands Used

```bash
enable
configure terminal
interface gigabitEthernet0/0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

### What These Commands Accomplish

- **`enable`**  
  Elevates the session from *user EXEC* (`Router>`) to *privileged EXEC* (`Router#`).  
  This is required to run administrative commands such as entering configuration mode or inspecting routing tables.

- **`configure terminal`**  
  Enters *global configuration mode*, enabling system-wide configuration changes.

- **`interface gigabitEthernet0/0/0`**  
  Selects the router’s physical port. Cisco names interfaces by:  
  `type / slot / subslot / port`.

- **`ip address <ip> <mask>`**  
  Assigns a Layer 3 address and subnet mask to the interface.  
  This makes the router reachable on that subnet.

- **`no shutdown`**  
  Activates the interface. Cisco router interfaces default to *administratively down*, so this command is essential.

### Engineering Note  
The temporary address `192.168.1.1` was used only to activate and test the interface. VLAN subnets will be applied later using subinterfaces.

---

# 3. DHCP Configuration for VLAN 10

The router provides dynamic IP assignments using a DHCP pool dedicated to VLAN 10.

![Step 2]("images/Step 2.png")

### Commands Used

```bash
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

### What These Commands Accomplish

- **`ip dhcp excluded-address`**  
  Prevents DHCP from assigning important addresses.  
  `.1` is reserved as the gateway; `.10` is held for future static use.

- **`ip dhcp pool VLAN10_Pool`**  
  Creates a DHCP pool named after the VLAN for clarity.

- **`network`**  
  Specifies the subnet and mask that the pool is allowed to lease.

- **`default-router`**  
  Specifies the gateway that DHCP clients will use.  
  This must match the router’s future VLAN10 subinterface.

- **`dns-server`**  
  Distributes Google’s public DNS server to clients.

### Engineering Note  
DHCP pools are mapped to VLANs by matching the **gateway IP** and the **network statement**, not by their names.

---

# 4. VLAN Creation & Access Port Assignment on the Switch

We configured VLAN segmentation and assigned interfaces to the appropriate VLAN.

![Step 3]("images/Step 3.png")

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

### What These Commands Accomplish

- **`vlan 10` / `vlan 20`**  
  Creates logical Layer 2 broadcast domains.

- **`switchport mode access`**  
  Configures Fa0/2 to carry untagged frames for one VLAN.

- **`switchport access vlan 10`**  
  Places PC0 into VLAN 10’s broadcast domain.

- **`no shutdown`**  
  Ensures the port is enabled physically and logically.

### Simple Troubleshooting Tip  
If the PC never receives a DHCP address, verify Fa0/2 is assigned to the **correct VLAN**.

---

# 5. Layer 2 Verification (VLANs and Trunks)

We validated the switch’s VLAN membership, trunking capability, and port status.

![Switch State]("images/Step 5 vlan brief interfaces trunk Switch.png")

### Commands Used

```bash
show vlan brief
show interfaces trunk
show ip interface brief
```

### What We Confirmed

- VLAN 10 exists and contains port Fa0/2  
- Fa0/1 is capable of operating as a trunk port  
- All interfaces are up and functioning  

---

# 6. Configuring Router-on-a-Stick (Subinterfaces)

We implemented inter-VLAN routing by creating subinterfaces for VLAN 10 and VLAN 20.

![Subinterfaces]("images/Step 1.png")

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

### What These Commands Accomplish

- **`interface g0/0/0.10`**  
  Establishes a logical subinterface for VLAN 10 on the physical link.

- **`encapsulation dot1Q 10`**  
  Enables 802.1Q tagging so the router can distinguish VLAN 10 frames.

- **`ip address`**  
  Assigns the VLAN’s default gateway.  
  VLAN10 → `192.168.10.1`  
  VLAN20 → `192.168.20.1`

### Engineering Note  
Router-on-a-stick is required when the switch is Layer 2-only and cannot route between VLANs on its own.

---

# 7. Configuring the Trunk Link Between Router and Switch

![Trunk]("images/Step 5 vlan brief interfaces trunk Switch.png")

### Commands Used

```bash
interface fa0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown
```

### What These Commands Accomplish

- **`switchport mode trunk`**  
  Converts the interface into an 802.1Q trunk capable of carrying multiple VLANs.

- **`allowed vlan all`**  
  Permits VLAN10 and VLAN20 to traverse the trunk.

### Simple Troubleshooting Tip  
If VLAN traffic does not reach the router, ensure the switch port is set to **trunk**, not **access**.

---

# 8. PC DHCP Configuration

![PC DHCP]("images/Step 4.png")

The PC was configured for DHCP and automatically received:

- IP address: `192.168.10.2`  
- Gateway: `192.168.10.1`  
- Mask: `255.255.255.0`  
- DNS: `8.8.8.8`  

All values match the router’s VLAN10 subinterface and DHCP pool.

---

# 9. Final Connectivity Validation

![Successful Ping]("images/Step 5 Ping VLANs.png")

### What Was Tested

- **Ping to 192.168.10.1**  
  Confirms PC’s default gateway and VLAN10 routing path.

- **Ping to 192.168.20.1**  
  Validates inter-VLAN routing through the router.

![Bindings]("images/Step 5 dhcp binding Router.png")

`show ip dhcp binding` verified that the router successfully leased `192.168.10.2` to PC0.

---

# Deep Command Reference

### EXEC & Global Configuration
- `enable` → privileged mode  
- `configure terminal` → global configuration mode  

### Router Interfaces
- `interface g0/0/0` → physical interface  
- `interface g0/0/0.X` → logical VLAN subinterface  
- `encapsulation dot1Q X` → sets VLAN tagging  
- `ip address X Y` → assigns Layer 3 address  
- `no shutdown` → activates interface  

### DHCP
- `ip dhcp excluded-address` → reserves IPs  
- `ip dhcp pool` → creates a DHCP pool  
- `network` → defines DHCP subnet  
- `default-router` → client gateway  
- `dns-server` → client DNS  

### Switch VLANs
- `vlan X` → creates VLAN  
- `switchport mode access` → access port  
- `switchport access vlan X` → assigns VLAN  

### Trunking
- `switchport mode trunk` → enables tagging  
- `allowed vlan all` → passes all VLANs  

### Verification
- `show vlan brief` → VLAN list  
- `show interfaces trunk` → trunk details  
- `show ip dhcp binding` → leased IPs  

---

# Summary

This project implemented a fully segmented multi-VLAN network using DHCP and router-on-a-stick routing. By configuring VLANs on a Layer 2 switch, setting up subinterfaces with 802.1Q encapsulation, enabling trunking, and deploying DHCP pools mapped to each VLAN, the network achieved dynamic addressing and inter-VLAN communication. Each command was chosen deliberately to illustrate how Layer 2 segmentation
