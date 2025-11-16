# Basic Routed VLAN Lab in Cisco Packet Tracer  
Single-Router, Single-Switch, Single-PC Topology with DHCP & Router-on-a-Stick

[![Cisco IOS](https://img.shields.io/badge/Platform-Cisco%20IOS-1ba0d7?logo=cisco)](https://www.cisco.com/)
[![Packet Tracer](https://img.shields.io/badge/Tool-Packet%20Tracer-0a7cff?logo=cisco)](https://www.netacad.com/courses/packet-tracer)
[![Networking](https://img.shields.io/badge/Focus-Routing_&_Switching-green)](#)
[![DHCP](https://img.shields.io/badge/Feature-DHCP_Automation-yellow)](#)
[![VLANs](https://img.shields.io/badge/Feature-VLAN10_Marketing-orange)](#)

---

## Project Overview

This lab walks through the complete configuration of a **routed VLAN access network** using Cisco Packet Tracer. A single router provides default-gateway and DHCP services for a Marketing VLAN, while a Layer 2 switch handles VLAN tagging and access/trunk ports, and a PC functions as a DHCP client in the segmented subnet. The goal is to demonstrate end-to-end understanding of VLAN creation, trunking, router-on-a-stick subinterfaces, DHCP scopes, and basic connectivity verification using ICMP.

All configuration and troubleshooting steps are documented here so the lab can be reproduced from scratch or used as a study artifact. The corresponding Packet Tracer file is included in the repository for cloning or download:  

**[Network Segmentation & VLAN Configuration](Network%20Segmentation%20%26%20VLAN%20Configuration.pkt)**

---

## 1. Lab Topology & Design

### 1.1 Logical Topology

The entire lab uses a minimal three-device design:

- **Router0** - default gateway and DHCP server  
- **Switch0** - Layer 2 access switch with VLANs and one trunk  
- **PC0** - endpoint in the Marketing VLAN

<img width="1920" height="1032" alt="S1" src="https://github.com/user-attachments/assets/6464d5e5-b183-4bd6-acf2-0c06aa920bdc" />

> **Cabling**  
> - Router `Gig0/0/0` ↔ Switch `Fa0/1`: *copper straight-through* (router-to-switch)  
> - Switch `Fa0/2` ↔ PC `Fa0`: *copper straight-through* (switch-to-PC)  

Even in a small lab, explicitly choosing correct cable types reinforces good physical-layer habits.

### 1.2 IP & VLAN Plan

| VLAN | Name       | Subnet            | Gateway (Router)    | Notes                            |
|------|------------|-------------------|---------------------|----------------------------------|
| 10   | Marketing  | `192.168.10.0/24` | `192.168.10.1/24`   | Active, used by PC0              |
| 20   | HR         | (reserved)        | (future expansion)  | Created but not used in this lab |

DHCP for VLAN 10:

- Scope: `192.168.10.0/24`  
- Excluded: `192.168.10.1`, `192.168.10.10`  
- Default gateway: `192.168.10.1`  
- DNS server: `8.8.8.8`

---

## 2. Switch Configuration - VLANs & Trunking

The switch provides Layer 2 segmentation and a single 802.1Q trunk toward the router.

### 2.1 VLAN Creation

VLANs were created and named for clarity:

```cisco
vlan 10
 name Marketing
vlan 20
 name HR
```

<img width="702" height="712" alt="S5 2_Results" src="https://github.com/user-attachments/assets/92f6362f-e833-4234-b1bd-54431269aa0c" />

> The `show vlan brief` output confirms VLAN 10 (`Marketing`) is active and that its access port is `Fa0/2`. VLAN 20 exists for future HR endpoints but is not yet associated with any ports.

### 2.2 Access & Trunk Port Mapping

- **Access port (client-facing):**  
  - `Fa0/2` is configured as an access port in VLAN 10.

- **Trunk port (router-facing):**  
  - `Fa0/1` is configured as an 802.1Q trunk and allowed to carry VLANs 10 and 20.

```cisco
interface fastEthernet0/2
 switchport mode access
 switchport access vlan 10

interface fastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

<img width="702" height="712" alt="S4" src="https://github.com/user-attachments/assets/86cc67d5-d1ef-4483-a912-b269892fa2f5" />

> This closely mirrors how an access switch in a production network connects user devices on access ports while uplinking to a router or L3 switch via a trunk.

### 2.3 Switch Interface Health

<img width="702" height="712" alt="S5 1_Results" src="https://github.com/user-attachments/assets/4c250d80-3b1c-4d79-8a8f-a0b20ad83a08" />

> The `show ip interface brief` output shows `Fa0/1` and `Fa0/2` in an `up/up` state, while all other interfaces are administratively down. The switch itself does not require an IP address for this lab; it operates purely at Layer 2.

---

## 3. Router Configuration - Subinterfaces & DHCP

### 3.1 Initial Physical Interface Setup

The first step was to bring up the router’s physical interface toward the switch:

```cisco
interface gigabitEthernet0/0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

<img width="702" height="712" alt="S2" src="https://github.com/user-attachments/assets/281c39f7-c541-4a2f-bac3-fe7b95c7a8f8" />

This provided basic connectivity on `Gig0/0/0`, but as later troubleshooting showed, the `192.168.1.0/24` network did **not** match the DHCP scope being created for `192.168.10.0/24`.

### 3.2 DHCP Configuration

DHCP was configured on the router with a dedicated pool for VLAN 10:

```cisco
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

<img width="702" height="712" alt="S3" src="https://github.com/user-attachments/assets/1fcb5528-5380-4c43-b51e-ac02c4b7ac4e" />

> `show ip dhcp binding` was run from global configuration mode (`Router(config)#`), which produced an “invalid input” error. Exiting back to privileged EXEC (`Router#`) resolved the issue and allowed verification of DHCP bindings later in the lab.

### 3.3 Router-on-a-Stick Subinterface

To correctly route for VLAN 10, a subinterface was created on `G0/0/0`:

```cisco
interface gigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

![Subinterface for VLAN 10](<S8.png>)

> This subinterface terminates the Marketing VLAN and acts as the default gateway for all hosts in `192.168.10.0/24`. The encapsulation command ensures that frames tagged with VLAN 10 arriving on the trunk are mapped to this subinterface.

---

## 4. End-Host Configuration & Final State

### 4.1 DHCP Client on PC0

Once the router’s DHCP and subinterface configuration were complete and the switch ports were correctly mapped, PC0 was set to obtain an IP address automatically.

<img width="702" height="712" alt="S9" src="https://github.com/user-attachments/assets/224756e5-3c37-4b0c-b278-8e95724f283c" />

The PC successfully received:

- `IPv4 Address`: `192.168.10.2`  
- `Subnet Mask`: `255.255.255.0`  
- `Default Gateway`: `192.168.10.1`

A ping test from PC0 to the router gateway (`192.168.10.1`) showed 0% packet loss, confirming full Layer 2 and Layer 3 connectivity between the client and the router.

---

## 5. Troubleshooting Walkthrough (In Order)

This lab intentionally preserves the troubleshooting sequence to reflect realistic problem-solving rather than a “perfect first try.”

### 5.1 Initial Build & Failed Ping

After wiring the devices and performing the initial configurations, the first verification step was a ping from PC0 to the router’s IP address. At that point, the router’s physical interface `G0/0/0` was configured with `192.168.1.1/24`, and the switch trunk/access settings were still being finalized.

<img width="702" height="712" alt="S6" src="https://github.com/user-attachments/assets/bd008a75-7fc2-430a-9ad5-0154fa86655b" />

- PC0 attempted to ping `192.168.1.1` and received 100% packet loss.  
- A follow-up ping from the switch to the same address also failed, indicating the issue was not limited to the end host.

### 5.2 Verifying Layer 2 - VLANs & Trunk

The next step was to verify the Layer 2 configuration:

1. Checked VLAN assignments with `show vlan brief` to ensure `Fa0/2` belonged to VLAN 10.  
2. Verified trunk status with `show interfaces trunk` to confirm `Fa0/1` was operating as an 802.1Q trunk and carrying VLAN 10.  
3. Reviewed switch interface status via `show ip interface brief` to confirm link state (`up/up`) on `Fa0/1` and `Fa0/2`.

These commands confirmed that Layer 2 was configured correctly, which pushed the investigation toward IP addressing and routing.

### 5.3 Identifying the IP/Subnet Mismatch

While reviewing the router configuration and DHCP pool, a key mismatch stood out:

- Router interface: `192.168.1.1/24` on `G0/0/0`  
- DHCP pool: `192.168.10.0/24` with default gateway `192.168.10.1`

This meant:

- The router had **no interface** in the `192.168.10.0/24` network.  
- Even if clients received DHCP addresses, they would not have a valid gateway reachable over the trunk.

### 5.4 Implementing Router-on-a-Stick & Retesting

To fix the mismatch and properly route for VLAN 10:

1. Created subinterface `G0/0/0.10` with `encapsulation dot1Q 10` and IP `192.168.10.1/24`.  
2. Confirmed the subinterface was `up/up`.  
3. Ensured the switch port to the router (`Fa0/1`) was in trunk mode and allowed VLAN 10.  
4. Set PC0 to use DHCP so it could obtain a `192.168.10.x` address and the correct default gateway (`192.168.10.1`).

<img width="702" height="712" alt="S7" src="https://github.com/user-attachments/assets/00026181-859b-4b83-b092-f29eb405147f" />

<img width="702" height="712" alt="S8" src="https://github.com/user-attachments/assets/2282e1c5-b1ce-48be-a4ef-652cad6b3023" />

With these changes in place, a new ping from PC0 to `192.168.10.1` succeeded with 0% loss, and `ipconfig` on the PC showed a leased IP of `192.168.10.2` with the expected subnet mask and gateway.

---

## 6. Quick Reference - Core Commands

### 6.1 Switch - VLANs, Access & Trunk

```cisco
! VLAN creation
vlan 10
 name Marketing
vlan 20
 name HR

! Access port for Marketing VLAN
interface fastEthernet0/2
 switchport mode access
 switchport access vlan 10

! Trunk to Router
interface fastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

### 6.2 Router - Subinterface & DHCP

```cisco
! Router-on-a-stick for VLAN 10
interface gigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

! DHCP configuration for VLAN 10
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

---

By the end of this lab, the router, switch, and PC together form a complete example of **VLAN-based segmentation with router-on-a-stick and DHCP**—exactly the pattern used in many small office and lab environments. The `.pkt` file in the repository can be opened directly in Packet Tracer to explore or modify the configuration further.
