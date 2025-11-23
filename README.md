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

![Topology]("images/PacketTracer_Topology_1.png")

---

# 1. Logical Topology Description

The physical connections in this lab consist of:

- **Router0 GigabitEthernet0/0/0 ↔ Switch0 FastEthernet0/1** (trunk link)  
- **Switch0 FastEthernet0/2 ↔ PC0** (access link in VLAN 10)  

This topology mirrors a traditional single-switch, single-router environment in which the router is responsible for routing between VLANs because the switch operates at Layer 2.

---

# 2. Router Initialization & Physical Interface Setup

When the router boots, IOS presents the System Configuration Dialog. We bypass it to configure the device manually from privileged EXEC mode.

![Router Init]("images/Step 1.png")

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

![DHCP Config]("images/Step 2.png")

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

![VLAN Config]("images/Step 3.png")

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

![VLAN & Trunk Verification]("images/Step 5 vlan brief interfaces trunk Switch.png")

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

![Trunk Config]("images/Step 5 vlan brief interfaces trunk Switch.png")

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

![PC DHCP]("images/Step 4.png")

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

![DHCP Bindings]("images/Step 5 dhcp binding Router.png")

`show ip dhcp binding` confirmed the DHCP lease for `192.168.10.2`.

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

