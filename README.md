# Network Segmentation & VLAN Configuration  
Router-on-a-Stick • DHCP • VLANs • Cisco Packet Tracer

[![Cisco IOS](https://img.shields.io/badge/Platform-Cisco_IOS-1ba0d7?logo=cisco)](https://www.cisco.com/)
[![Packet Tracer](https://img.shields.io/badge/Tool-Packet_Tracer-0a7cff?logo=cisco)](https://www.netacad.com/courses/packet-tracer)
[![Routing & Switching](https://img.shields.io/badge/Focus-Routing_&_Switching-green)](#)
[![DHCP](https://img.shields.io/badge/Service-DHCP_Automation-yellow)](#)
[![VLANs](https://img.shields.io/badge/Feature-VLAN10-orange)](#)

---

## Project Overview

This project demonstrates how to configure a segmented Layer 2 network with DHCP, VLANs, trunking, and router-on-a-stick using Cisco Packet Tracer. A single router provides Layer 3 services for the Marketing VLAN, while a Layer 2 switch handles VLAN membership and 802.1Q trunking. The PC in VLAN 10 receives its network configuration dynamically via a DHCP pool hosted on the router.

The goal is not only to show the final working configuration, but also to walk through the real troubleshooting steps and decisions made along the way; mirroring how network engineers diagnose and fix issues in live environments. You can download and explore the completed Packet Tracer file below:

**[Network Segmentation & VLAN Configuration](Network%20Segmentation%20%26%20VLAN%20Configuration.pkt)**

---

## 1. Logical Topology & Planning

The environment contains:

- **Router0** → Provides routing and DHCP  
- **Switch0** → Handles VLANs and trunking  
- **PC0** → Client in VLAN 10, set to obtain IP via DHCP

<img width="1920" height="1032" alt="S1" src="https://github.com/user-attachments/assets/3d50010f-e3a6-474e-99cf-e3b490624148" />

> **Cabling Used:**  
> - Router ↔ Switch → Straight-through (different device types)  
> - Switch ↔ PC → Straight-through  

**VLAN / IP Design**

| VLAN | Purpose    | Subnet           | Gateway         |
|------|-------------|------------------|-----------------|
| 10   | Marketing   | 192.168.10.0/24  | 192.168.10.1    |
| 20   | HR          | (Reserved)       | (Planned)       |

---

## 2. Router Initial Configuration (S2)

Upon opening the router, Packet Tracer displays the **System Configuration Dialog**.  
We intentionally selected **“no”** so that we could build the configuration manually.

<img width="702" height="712" alt="S2" src="https://github.com/user-attachments/assets/a272acdd-4ba3-4baa-9d69-837850ebb2d6" />

We entered privileged EXEC mode using:

- **`enable`** → Grants access to advanced commands  
- **`configure terminal`** → Enters global configuration mode  

The physical interface `Gig0/0/0` was initially configured with `192.168.1.1`.  
Although this brought the interface up, it did **not** match our intended network (`192.168.10.0/24`), which would later cause the first ping failure.

---

## 3. DHCP Configuration (S3)

With the router online, we configured DHCP exclusions and created a dedicated pool for VLAN 10.

<img width="702" height="712" alt="S3" src="https://github.com/user-attachments/assets/bb0eca13-c95c-4a8b-92b4-7f913e978cfa" />

We excluded `.1` (the gateway) and `.10` (reserved), defined the network, and then verified bindings. The first verification attempt failed because the command was run inside configuration mode; an IOS nuance that provided a helpful reminder:

> **Execution commands only run from privileged EXEC mode (`Router#`).**

Once in the correct mode, DHCP bindings displayed properly.

---

## 4. VLAN Creation & Port Assignment (S4)

Next, we configured the switch to support segmented Layer 2 boundaries.

<img width="702" height="712" alt="S4" src="https://github.com/user-attachments/assets/c35f99c2-5b3a-4478-bee2-05d7dc77b86d" />

We created two VLANs:

- **VLAN 10 → Marketing**  
- **VLAN 20 → HR**

Port assignments:

- `Fa0/2` → Access port for VLAN 10  
- `Fa0/1` → Uplink toward the router  

At this stage the switch’s VLAN database and port assignments were correct, but trunking and the router subinterface had not yet been configured.

---

## 5. L2 Verification (S5.1, S5.2, S5.3)

Before testing connectivity, we verified the switch’s operational state through three key commands.

### **Switch Interface Summary (S5.1)**  
<img width="702" height="712" alt="S5 1_Results" src="https://github.com/user-attachments/assets/366fbc36-eaa0-4ec5-993c-5e0ff537c04b" />

`Fa0/1` and `Fa0/2` are up, confirming clean physical connectivity.

### **VLAN Table (S5.2)**  
<img width="702" height="712" alt="S5 2_Results" src="https://github.com/user-attachments/assets/0ee564ff-409f-476a-8507-14c423e9d33c" />

VLAN 10 correctly lists `Fa0/2`.

### **Trunk Capability (S5.3)**  
<img width="702" height="712" alt="S5 3_Results" src="https://github.com/user-attachments/assets/595de571-e9a9-4c02-ae23-e35a7f591a10" />

`Fa0/1` is ready to become a trunk but is not fully configured yet.

---

## 6. First Connectivity Test – Ping Failure (S6)

With everything appearing correct electrically, we attempted a ping to the router.  
The PC could not reach `192.168.1.1`.

<img width="702" height="712" alt="S6" src="https://github.com/user-attachments/assets/0187f75c-d972-45a4-bd2b-62ece95726a7" />

This was the moment we realized there was a mismatch between:

- the router’s configured IP (`192.168.1.1`)  
- the DHCP scope (`192.168.10.x`)  
- the VLAN design (VLAN 10 should use `192.168.10.1`)  

This inconsistency guided the next steps in troubleshooting.

---

## 7. Creating the Router-on-a-Stick Subinterface (S7)

To align routing with our VLAN design, we created a **subinterface** for VLAN 10 and applied 802.1Q encapsulation.

<img width="702" height="712" alt="S7" src="https://github.com/user-attachments/assets/33e33ee1-949d-4b00-9366-cffe12432720" />

This assigned the correct gateway:

- `Gig0/0/0.10 → 192.168.10.1`  
- Tagged to VLAN 10  
- Matched to the DHCP configuration  

This was the turning point in fixing the mismatch causing the ping failure.

---

## 8. Updating the Switch Trunk Configuration (S8)

Next, we returned to the switch to complete the uplink configuration so that VLAN-tagged frames could reach the router.

<img width="702" height="712" alt="S8" src="https://github.com/user-attachments/assets/24e459a1-c2cd-4fd8-b6a2-b97011a127a8" />

We set `Fa0/1` to trunk mode and allowed VLANs 10 and 20.  
This established full Layer 2 communication between the VLAN and the router subinterface.

---

## 9. Successful Ping & DHCP Assignment (S9)

With the gateway corrected and trunking finalized, the PC finally obtained a proper DHCP lease.

<img width="702" height="712" alt="S9" src="https://github.com/user-attachments/assets/f81f64ea-c118-4300-9e59-35992b2d6003" />

- Assigned IP: `192.168.10.2`  
- Default Gateway: `192.168.10.1`  
- DNS: `8.8.8.8`  

A ping to `192.168.10.1` returned **100% success**, confirming the network was now fully functional.

---

## 10. Final PC Configuration Confirmation (S10)

This image shows the PC set explicitly to **DHCP**, which is required for it to receive the correct addressing information.

<img width="702" height="712" alt="S10" src="https://github.com/user-attachments/assets/363e828e-3060-42db-bb27-98185406943a" />

> If the PC had been left in static mode earlier in testing, this alone would have blocked DHCP. Ensuring it was set to DHCP was essential for final validation.

---

## Conclusion

This project demonstrated a complete workflow for building and validating a segmented network with VLANs, DHCP, and router-on-a-stick routing. Beyond applying commands, it walked through real troubleshooting steps: identifying addressing mismatches, correcting subinterfaces, enabling trunking, and validating end-to-end connectivity. The final working environment behaves exactly like a small enterprise network, with scalable VLAN architecture and centralized Layer 3 routing.

