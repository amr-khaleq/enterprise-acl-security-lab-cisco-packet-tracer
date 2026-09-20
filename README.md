# Enterprise Network Traffic Filtering with Cisco IOS Access Control Lists (ACLs)

![Network Security](https://img.shields.io/badge/Security-Access%20Control%20Lists-red)
![Cisco](https://img.shields.io/badge/Vendor-Cisco%20IOS-blue)
![Architecture](https://img.shields.io/badge/Design-Router--on--a--Stick-green)
![Simulation](https://img.shields.io/badge/Simulator-Packet%20Tracer-orange)

## 📌 Executive Summary
This enterprise-grade lab simulates a multi-department network implementing granular access policies using **Named Extended Access Control Lists (ACLs)**. In modern enterprise environments, inter-VLAN segmentation is insufficient without Layer 3/Layer 4 traffic filtering. 

This project demonstrates the design, deployment, verification, and troubleshooting of traffic filtering policies on a **Router-on-a-Stick** topology using Cisco Packet Tracer.

---

## 🏗 Network Topology & Architecture

```text
 <img width="1848" height="818" alt="Screenshot 2026-09-21 002704" src="https://github.com/user-attachments/assets/c81ce397-43ba-43bd-aec4-af4e064a8ab6" />

```

---

## 📋 Addressing & VLAN Schema

| Department / Role | VLAN ID | Subnet CIDR | Default Gateway | Assigned Hosts / IPs |
| :--- | :--- | :--- | :--- | :--- |
| **Sales** | 10 | `192.168.10.0/24` | `192.168.10.1` | `SALES-PC` (192.168.10.10) |
| **Human Resources** | 20 | `192.168.20.0/24` | `192.168.20.1` | `HR-PC` (192.168.20.10) |
| **IT Support** | 30 | `192.168.30.0/24` | `192.168.30.1` | `IT-PC` (192.168.30.10) |
| **Server Farm** | 40 | `192.168.40.0/24` | `192.168.40.1` | `WEB-SRV` (192.168.40.10)<br>`DNS-SRV` (192.168.40.20) |

---

## 🛡 Security Matrix & Access Policy Requirements

| Source Segment | Target: SALES | Target: HR | Target: IT | Target: SERVERS | External / Any |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **SALES (VLAN 10)** | Local | ❌ Deny | ❌ Deny | ✅ Permit (L3/L4) | ✅ Permit |
| **HR (VLAN 20)** | ❌ Deny | Local | ❌ Deny | ✅ Permit (L3/L4) | ✅ Permit |
| **IT (VLAN 30)** | ✅ Permit | ✅ Permit | Local | ✅ Permit | ✅ Permit |

---

## ⚙️ Device Configurations

### 1. Switch Configuration (`SW1`)
```ios
enable
configure terminal
hostname SW1

! Create VLANs
vlan 10
 name SALES
vlan 20
 name HR
vlan 30
 name IT
vlan 40
 name SERVERS
exit

! Access Port Allocations
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
 no shutdown

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 20
 no shutdown

interface FastEthernet0/4
 switchport mode access
 switchport access vlan 30
 no shutdown

interface range FastEthernet0/5 - 6
 switchport mode access
 switchport access vlan 40
 no shutdown

! 802.1Q Trunk Link to Router
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 no shutdown
exit
```

### 2. Router Subinterfaces (`R1`)
```ios
enable
configure terminal
hostname R1

interface GigabitEthernet0/0
 no ip address
 no shutdown
exit

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
exit

interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
exit
```

### 3. Named Extended ACL Implementation
Standard practice recommends applying Extended ACLs as close to the source as possible with inbound direction (`in`).

```ios
! -------------------------------
! Sales Inbound Filter
! -------------------------------
ip access-list extended SALES-FILTER
 remark Block access to internal departments
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 remark Permit access to corporate servers
 permit ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255
 remark Permit all other external traffic
 permit ip 192.168.10.0 0.0.0.255 any
exit

! -------------------------------
! HR Inbound Filter
! -------------------------------
ip access-list extended HR-FILTER
 remark Block access to IT infrastructure
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 remark Permit access to corporate servers
 permit ip 192.168.20.0 0.0.0.255 192.168.40.0 0.0.0.255
 remark Permit remaining authorized traffic
 permit ip 192.168.20.0 0.0.0.255 any
exit

! -------------------------------
! IT Inbound Filter (Unrestricted)
! -------------------------------
ip access-list extended IT-FILTER
 remark Unrestricted administrative access
 permit ip 192.168.30.0 0.0.0.255 any
exit

! -------------------------------
! Apply Filters to Subinterfaces
! -------------------------------
interface GigabitEthernet0/0.10
 ip access-group SALES-FILTER in
exit

interface GigabitEthernet0/0.20
 ip access-group HR-FILTER in
exit

interface GigabitEthernet0/0.30
 ip access-group IT-FILTER in
exit
```

---

## 🔍 Verification & Troubleshooting Guide

### Key CLI Verification Commands
```ios
! Verify configured rules and match counters
show ip access-lists

! Check which ACL is applied and in which direction
show ip interface gigabitEthernet 0/0.10 | include access list
```

### Verification Matrix
- `SALES-PC` $\rightarrow$ `HR-PC (192.168.20.10)`: **Destination Host Unreachable / Dropped** *(Matches SALES-FILTER line 10)*
- `SALES-PC` $\rightarrow$ `WEB-SRV (192.168.40.10)`: **Success (ICMP Reply / HTTP 200 OK)** *(Matches SALES-FILTER line 30)*
- `HR-PC` $\rightarrow$ `IT-PC (192.168.30.10)`: **Destination Host Unreachable / Dropped** *(Matches HR-FILTER line 10)*
- `IT-PC` $\rightarrow$ **All Subnets**: **Success (Full Inter-VLAN Reachability)**

---

## 💡 Key Design Rules & Lessons Learned
1. **Rule Specificity Ordering**: ACL statements are evaluated sequentially from top to bottom. Specific `deny` rules must precede general `permit any` rules to avoid policy bypass.
2. **Implicit Deny Behavior**: Every Cisco ACL terminates with an invisible `deny ip any any`. Traffic that does not match an explicit rule is dropped automatically.
3. **Inbound vs. Outbound Placement**: Filtering inbound (`in`) at the subinterface level drops unauthorized packets before routing engine processing, conserving router CPU and memory.
