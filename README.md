# Basic Routed VLAN Lab in Cisco Packet Tracer  
Single-Router, Single-Switch, Single-PC Topology with DHCP & Router-on-a-Stick

[![Cisco IOS](https://img.shields.io/badge/Platform-Cisco%20IOS-1ba0d7?logo=cisco)](https://www.cisco.com/)
[![Packet Tracer](https://img.shields.io/badge/Tool-Packet%20Tracer-0a7cff?logo=cisco)](https://www.netacad.com/courses/packet-tracer)
[![Networking](https://img.shields.io/badge/Focus-Routing_&_Switching-green)](#)
[![DHCP](https://img.shields.io/badge/Feature-DHCP_Automation-yellow)](#)
[![VLANs](https://img.shields.io/badge/Feature-VLAN10_Marketing-orange)](#)

---

## Project Overview

This mini-lab demonstrates how to build a **routed VLAN environment** in Cisco Packet Tracer using:

- A single **router** performing **router-on-a-stick** for VLAN 10  
- A **Layer 2 switch** providing access and trunk ports  
- A **PC** receiving its IPv4 address from a **DHCP pool**  
- Proper use of **VLANs, trunks, subinterfaces, and DHCP** to simulate a segmented enterprise access network

Even though it’s a small topology, the lab walks through the same concepts used in real networks:

- Choosing the correct **cable types**  
- Mapping **physical ports** to **logical VLANs**  
- Creating a **DHCP pool** and verifying bindings  
- Troubleshooting CLI mistakes and mode errors

> This README is meant to stand alone as the “project file” for GitHub, combining design decisions, configuration, and troubleshooting notes into one documented lab.

---

## 1. Lab Topology & Design

### 1.1 High-Level Logical Diagram

The final logical design:

- **VLAN 10 – Marketing**  
  - Subnet: `192.168.10.0/24`  
  - Default gateway: `192.168.10.1` (router subinterface `G0/0/0.10`)  
  - DHCP range: `192.168.10.2 – 192.168.10.254`  
  - Excluded: `192.168.10.1`, `192.168.10.10`

Devices used:

| Device   | Role                   | Key Interface     | IP / VLAN                         |
|----------|------------------------|-------------------|-----------------------------------|
| Router0  | Default gateway + DHCP | `G0/0/0.10`       | `192.168.10.1/24`, VLAN 10        |
| Switch0  | L2 Access & Trunk      | `Fa0/1` (trunk)   | VLANs 1, 10, 20                   |
|          |                        | `Fa0/2` (access)  | Access port in VLAN 10            |
| PC0      | Client endpoint        | `Fa0`             | DHCP address in `192.168.10.0/24` |

### 1.2 Packet Tracer View

![Logical Topology](<S3.png>)

> **Cabling choices:**  
> - Router `G0/0/0` ↔ Switch `Fa0/1`: *copper straight-through* (different device types)  
> - Switch `Fa0/2` ↔ PC `Fa0`: *copper straight-through*  

Even small labs are a good place to form the habit of choosing **correct cable types** and documenting which ports are used.

---

## 2. Switch Configuration – VLANs & Trunking

The switch is responsible for local Layer 2 segmentation and forwarding frames toward the router.

### 2.1 VLAN Creation

We created two VLANs even though only VLAN 10 is actively used in this lab:

- VLAN 10 – `Marketing`  
- VLAN 20 – `HR` (reserved for future use)

![VLAN Table](<S1.png>)

> From the `show vlan brief` output we can see:
> - VLAN 10 (`Marketing`) is active and bound to **port `Fa0/2`**  
> - VLAN 20 exists but has no access ports assigned yet  
> - All unused ports remain in VLAN 1 (`default`)

Relevant CLI (simplified):

```text
vlan 10
 name Marketing
vlan 20
 name HR
```

### 2.2 Access & Trunk Port Mapping

![Interface/Trunk Status](<S2.png>)

- **Access Port:**  
  - `Fa0/2` → `switchport mode access`  
  - `switchport access vlan 10`  

- **Trunk Port:**  
  - `Fa0/1` (toward router)  
  - `switchport mode trunk`  
  - `switchport trunk allowed vlan 10,20`  

This gives us a classic **“router-on-a-stick”** setup: all VLAN traffic is funneled over a single 802.1Q trunk to the router.

> **Why this matters:**  
> Router-on-a-stick is commonly used in labs and smaller deployments where a single router interface must route traffic for multiple VLANs without needing additional physical ports.

### 2.3 Switch Interface Health

![Switch IP Interface Brief](<S5.2_Results.png>)

> `show ip interface brief` confirms:  
> - `Fa0/1` and `Fa0/2` are **up/up**  
> - All other ports are administratively down and unused  
> - The switch itself does not need an IP for this lab; it’s purely L2.

---

## 3. Router Configuration – Subinterfaces & DHCP

### 3.1 Physical Interface Bring-Up

Initially, the router’s physical interface `G0/0/0` was brought up with an IP on a different subnet (`192.168.1.1/24`):

![Initial Router Interface Config](<S2.png>)

Eventually, we shifted the routing logic onto a **subinterface** for VLAN 10 and treated the physical interface purely as a trunk parent.

### 3.2 Creating the DHCP Pool

DHCP configuration on the router:

```cisco
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

![DHCP Configuration & Binding](<S3.png>)

> **Error encountered:**  
> The first attempt to run `show ip dhcp binding` was done from *global config mode*, resulting in  
> `% Invalid input detected at '^' marker.`  
>  
> **Fix:** Exit back to privileged E```EC (`Router#`) and run the command there.  
> This reinforced the importance of understanding IOS **modes** when running commands.

### 3.3 Building the Router-on-a-Stick Subinterface

To match VLAN 10, we created **subinterface `G0/0/0.10`**:

```cisco
interface gigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

![Subinterface Configuration](<S6.png>)

> The subinterface serves as the **default gateway** for all hosts in VLAN 10.  
> With `encapsulation dot1Q 10`, the router can interpret 802.1Q tags coming from the switch trunk on `Fa0/1`.

---

## 4. End-Host Configuration & Testing

### 4.1 Client Obtains DHCP Lease

Once the router and switch were configured correctly, PC0 was set to obtain an IP address automatically.

![PC DHCP & Ping Success](<S9.png>)

The PC received:

- IPv4 Address: `192.168.10.2`  
- Subnet Mask: `255.255.255.0`  
- Default Gateway: `192.168.10.1`

This aligns exactly with the DHCP pool and the router’s subinterface.

> The PC uses **FastEthernet0**, connected to switch port `Fa0/2` with a straight-through copper cable.  
> Because `Fa0/2` belongs to VLAN 10 and `Fa0/1` is a trunk, packets are tagged and forwarded to `G0/0/0.10`, where they are routed.

### 4.2 Connectivity Tests

We validated connectivity in multiple ways:

1. **PC → Router default gateway**

   ```text
   ping 192.168.10.1
   ```  

2. **Router → PC**

   ```cisco
   ping 192.168.10.2
   ```  

3. **Switch → Router**

   ```cisco
   ping 192.168.10.1
   ```  

![Ping Attempts & Trunk Config](<S8.png>)

> Early pings to `192.168.1.1` failed (0% success) because that address no longer matched the VLAN 10 subnet after moving routing to `192.168.10.1`.  
> Updating the test target to `192.168.10.1` confirmed that Layer 2 and Layer 3 were correctly aligned.

---

## 5. Issues Encountered & How They Were Resolved

This lab intentionally left room for mistakes and fixes to mirror real troubleshooting.

### 5.1 CLI Mode Confusion – `show ip dhcp binding`

- **Symptom:**  
  - `% Invalid input detected at '^' marker.` when running `show ip dhcp binding`
- **Cause:**  
  - Command was executed in **global configuration mode** instead of **privileged E```EC**
- **Fix:**  
  - `end` or `Ctrl+Z` to return to `Router#`  
  - Re-run `show ip dhcp binding`

---

### 5.2 DHCP Pool Without a Matching Interface

- **Symptom:**  
  - DHCP pool defined for `192.168.10.0/24`, but no gateway interface in that subnet  
- **Cause:**  
  - Only `G0/0/0` with `192.168.1.1/24` existed at first  
- **Fix:**  
  - Created **subinterface** `G0/0/0.10` with `192.168.10.1/24` and proper dot1Q tag  
  - Verified with `show ip interface brief` and `show ip dhcp binding`

> This reflects a common real-world misconfiguration: DHCP scopes created for networks that the router isn’t actually attached to.

---

### 5.3 Trunk vs Access Port Alignment

- **Symptom:**  
  - Early connectivity issues and Packet Tracer simulation flags  
- **Cause:**  
  - Need to ensure:
    - `Fa0/1` is a **trunk** (802.1Q)  
    - `Fa0/2` is an **access port in VLAN 10**
- **Fix:**  
  - Explicitly configured:
    - `switchport mode trunk` on `Fa0/1`  
    - `switchport access vlan 10` on `Fa0/2`  

![Switch VLAN & Port Assignment](<S5.1_Results.png>)

---

### 5.4 Simulation Panel “```” Icons

- **Symptom:**  
  - In Simulation Mode, packets showed an “```” over the router and PC  
- **Reality:**  
  - Live pings and configs were correct; the topology worked  
- **Explanation:**  
  - Packet Tracer’s Simulation Mode can flag intermediate failures or legacy ARP lookups even when final connectivity is restored  
  - Since the lab objective is functional connectivity and correct configuration, we treated Simulation anomalies as **non-blocking** once ICMP tests passed

> It’s important to differentiate between **tool quirks** and **actual misconfigurations**.

---

## 6. Quick Reference – Core Commands

### 6.1 Switch – VLAN & Trunk

```cisco
vlan 10
 name Marketing
vlan 20
 name HR

interface fastEthernet0/2
 switchport mode access
 switchport access vlan 10

interface fastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

### 6.2 Router – Subinterface & DHCP

```cisco
! Router-on-a-stick for VLAN 10
interface gigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

! DHCP configuration
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.10

ip dhcp pool VLAN10_Pool
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

---

## 7. Takeaways

- Even a **three-device lab** exercises core enterprise skills:
  - VLAN design and port mapping  
  - Trunking and 802.1Q tagging  
  - Router-on-a-stick subinterfaces  
  - DHCP implementation and troubleshooting  
- Accurate documentation of **ports, cables, and commands** makes it easier to reproduce the lab and demonstrates attention to detail to anyone reviewing the project.

> This README can be dropped directly into a GitHub repo alongside the `.pkt` file to fully document the lab, the configuration decisions, and the troubleshooting process.

