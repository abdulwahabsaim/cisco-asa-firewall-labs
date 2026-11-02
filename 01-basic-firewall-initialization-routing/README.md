<div align="center">

# 🛡️ Cisco ASA Firewall: Interface Initialization, Security Zones & Enterprise Routing

[![Platform](https://img.shields.io/badge/Platform-Cisco_ASA_5520_v9.1(5)-00599C?style=for-the-badge&logo=cisco&logoColor=white)](https://cisco.com)
[![Software](https://img.shields.io/badge/Environment-EVE--NG_Community-1BA0D7?style=for-the-badge&logo=linux&logoColor=white)](https://eve-ng.net)
[![Security Zones](https://img.shields.io/badge/Security_Zones-Inside_(100)_|_Outside_(0)-brightgreen?style=for-the-badge)]()
[![Routing](https://img.shields.io/badge/Routing-Static_|_OSPFv2_|_EIGRP-blueviolet?style=for-the-badge)]()
[![Validation](https://img.shields.io/badge/Convergence-100%25_Verified-success?style=for-the-badge)]()

<p align="center">
  <b>Production-grade deployment and verification of a Cisco ASA 5520 Adaptive Security Appliance in EVE-NG. Covers the 4 pillars of ASA interface initialization, security level directionality (0-100), stateful ICMP inspection drop dynamics, inbound ACL traversal, static/default routing, and dynamic OSPFv2 vs. EIGRP Administrative Distance (AD) arbitration.</b>
</p>

</div>

---

## 📌 1. Executive Summary & Engineering Objective

In traditional Cisco IOS routing environments, bringing an interface into an operational forwarding state requires merely assigning an IPv4 address and un-shutting the physical port. In contrast, the **Cisco Adaptive Security Appliance (ASA)** operates as a dedicated stateful security barrier enforcing strict implicit traffic isolation.

This lab configures and validates the end-to-end initialization of a **Cisco ASA 5520 (Software Version 9.1(5))** bridging an untrusted transit network (`Outside`, Security Level `0`) and a trusted internal corporate segment (`Inside`, Security Level `100`).

### Core Engineering Objectives
1. **Four Pillars of Interface Activation:** Configure and verify `nameif`, `security-level`, `ip address`, and administrative activation (`no shutdown`).
2. **Security Level Trust Hierarchy & Traffic Directionality:** Audit default packet flow policies (High-to-Low permitted, Low-to-High denied).
3. **Stateful Inspection & Transit ICMP Troubleshooting:** Analyze the root cause of return packet drops for uninspected ICMP echo replies at Security Level 0, resolving reachability via inbound extended access-lists.
4. **ASA Routing Architecture:** Configure and test specific static routing (`route Outside <net> <mask> <gw>`), candidate default gateway routing (`route Outside 0.0.0.0 0.0.0.0 <gw>`), and dual dynamic routing protocols (OSPF Area 0 and EIGRP AS 1).
5. **Administrative Distance Arbitration:** Prove that the Cisco ASA RIB installs EIGRP routes (`D`, AD 90) over OSPF routes (`O`, AD 110) when matching prefixes are advertised simultaneously.

---

## 🗺️ 2. Network Topology & Architecture

The architecture consists of a Cisco ASA 5520 edge firewall, an upstream Cisco IOL Layer 3 router simulating an external transit provider and remote branch subnets, two intermediate Layer 2 bridging switches, and two inside VPC workstations.

<div align="center">
  <img src="topology.png" alt="Cisco ASA Basic Initialization & Routing Topology" width="460"/>
</div>

---

## 📊 3. IP Addressing & Security Zone Parameter Schema

| Device | Interface | Logical Name (`nameif`) | Security Level | IP Address | Subnet Mask | Function / Zone Role |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **ASA** | `Ethernet0` | `Outside` | `0` | `203.10.1.2` | `255.255.255.0` (`/24`) | Untrusted Transit Edge Uplink |
| **ASA** | `Ethernet1` | `Inside` | `100` | `10.1.1.2` | `255.255.255.0` (`/24`) | Trusted Internal Corporate LAN |
| **WAN-Router** | `Ethernet0/0` | N/A (IOS) | N/A | `203.10.1.1` | `255.255.255.0` (`/24`) | Upstream Provider Next-Hop Gateway |
| **WAN-Router** | `Loopback0` | N/A (IOS) | N/A | `22.2.2.2` | `255.255.255.0` (`/24`) | Simulated Remote Enterprise Branch A |
| **WAN-Router** | `Loopback1` | N/A (IOS) | N/A | `33.3.3.3` | `255.255.255.0` (`/24`) | Simulated Remote Enterprise Branch B |
| **WAN-Switch** | `Eth0/0 - Eth0/1` | N/A (IOS) | N/A | Unnumbered | Layer 2 Transit | Bridging ASA Eth0 to WAN-Router Eth0/0 |
| **LAN-Switch** | `Eth0/0 - Eth0/2` | N/A (IOS) | N/A | Unnumbered | Layer 2 Transit | Bridging ASA Eth1 to Inside End Hosts |
| **TestPC1** | `eth0` | N/A (VPCS) | N/A | `10.1.1.1` | `255.255.255.0` (`/24`) | Inside Workstation 1 (GW: 10.1.1.2) |
| **TestPC2** | `eth0` | N/A (VPCS) | N/A | `10.1.1.3` | `255.255.255.0` (`/24`) | Inside Workstation 2 (GW: 10.1.1.2) |

---

## 🛠️ 4. Cisco ASA & IOS Configuration Highlights

### 4.1 Cisco ASA 5520 Interface Initialization & Security Policies
```ios
hostname ASA
!
! Pillar 1-4: Physical and Logical Interface Activation
interface Ethernet0
 description WAN_Uplink_to_WAN-Switch
 nameif Outside
 security-level 0
 ip address 203.10.1.2 255.255.255.0
 no shutdown
!
interface Ethernet1
 description LAN_Downlink_to_LAN-Switch
 nameif Inside
 security-level 100
 ip address 10.1.1.2 255.255.255.0
 no shutdown
!
! Inbound Extended ACL permitting return transit traffic from Outside (Level 0 -> 100)
access-list 101 extended permit ip any any
access-group 101 in interface Outside
!
! Dynamic Routing: OSPF Area 0 (Standard Subnet Masks on ASA)
router ospf 1
 network 203.10.1.0 255.255.255.0 area 0
 log-adj-changes
!
! Dynamic Routing: EIGRP AS 1
router eigrp 1
 network 203.10.1.0 255.255.255.0
```

### 4.2 Cisco IOS WAN Router Configuration
```ios
hostname WAN-Router
!
interface Ethernet0/0
 description Downlink_to_WAN-Switch_facing_ASA_Outside
 ip address 203.10.1.1 255.255.255.0
 no shutdown
!
interface Loopback0
 description Simulated_Remote_Branch_Prefix_22
 ip address 22.2.2.2 255.255.255.0
 no shutdown
!
interface Loopback1
 description Simulated_Remote_Branch_Prefix_33
 ip address 33.3.3.3 255.255.255.0
 no shutdown
!
! Dynamic Routing: OSPF Area 0 (Inverse Wildcard Masks on IOS)
router ospf 1
 network 203.10.1.0 0.0.0.255 area 0
 network 22.2.2.0 0.0.0.255 area 0
 network 33.3.3.0 0.0.0.255 area 0
!
! Dynamic Routing: EIGRP AS 1
router eigrp 1
 network 203.10.1.0 0.0.0.255
 network 22.2.2.0 0.0.0.255
 network 33.3.3.0 0.0.0.255
 no auto-summary
```

<details>
<summary><b>📄 Click to expand full raw device configurations</b></summary>

* See [`asa-config.ios`](./asa-config.ios) for complete ASA running configuration.
* See [`wan-router-config.ios`](./wan-router-config.ios) for complete WAN router running configuration.
* See [`wan-switch-config.ios`](./wan-switch-config.ios) for WAN switch configuration.
* See [`lan-switch-config.ios`](./lan-switch-config.ios) for LAN switch configuration.

</details>

---

## 🔍 5. Verification & Operational Proof

### 5.1 Verification of Logical Interfaces and Security Zones
The `show nameif` command confirms that logical security zones and relative trust ratings are bound to physical hardware ports:
```text
ASA# show nameif
Interface                  Name                     Security
Ethernet0                  Outside                    0
Ethernet1                  Inside                   100
```

Validating Layer 3 operational state via `show interface ip brief`:
```text
ASA# show interface ip brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0                  203.10.1.2      YES manual up                    up
Ethernet1                  10.1.1.2        YES manual up                    up
```

---

### 5.2 ICMP Stateful Inspection Dynamics & Access List Validation
When `TestPC1` (`10.1.1.1`) pings `WAN-Router` (`203.10.1.1`), the ASA permits the outbound ICMP Echo Request (High-to-Low). However, because the ASA does not inspect ICMP statefully by default, return ICMP Echo Replies arriving at `Ethernet0` (Security Level 0) are dropped.

Enabling inbound extended ACL 101 permits transit return traffic, verified through incrementing access-list hit counters:
```text
ASA# show access-list
access-list 101; 1 elements; name hash: 0xe77d50b5
access-list 101 line 1 extended permit ip any any (hitcnt=5) 0x20676dfa
```

End-host verification on `TestPC1`:
```text
TestPC1> ping 203.10.1.1
203.10.1.1 icmp_seq=1 timeout
84 bytes from 203.10.1.1 icmp_seq=2 ttl=254 time=1.842 ms
84 bytes from 203.10.1.1 icmp_seq=3 ttl=254 time=1.921 ms
84 bytes from 203.10.1.1 icmp_seq=4 ttl=254 time=1.785 ms
84 bytes from 203.10.1.1 icmp_seq=5 ttl=254 time=1.810 ms
```

---

### 5.3 Routing Protocol Adjacency Verification

#### OSPF Neighbor Adjacency
```text
ASA# show ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
203.10.1.1        1   FULL/BDR        0:00:34     203.10.1.1      Outside
```

#### EIGRP Neighbor Adjacency
```text
ASA# show eigrp neighbors
EIGRP-IPv4 Neighbors for AS(1)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   203.10.1.1              Outside                  14 00:12:45   12   200  0  5
```

---

### 5.4 Administrative Distance (AD) Arbitration Analysis
When both OSPF Area 0 and EIGRP AS 1 are configured to advertise remote subnets `22.2.2.0/24` and `33.3.3.0/24`, the Cisco ASA evaluates the Administrative Distance of each routing source:
- **EIGRP Internal AD:** `90`
- **OSPF AD:** `110`

Executing `show route` confirms that EIGRP (`D`) successfully arbitrates over OSPF (`O`) in the ASA Routing Information Base (RIB):
```text
ASA# show route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

D    22.2.2.0 255.255.255.0 [90/156160] via 203.10.1.1, Outside
D    33.3.3.0 255.255.255.0 [90/156160] via 203.10.1.1, Outside
C    203.10.1.0 255.255.255.0 is directly connected, Outside
C    10.1.1.0 255.255.255.0 is directly connected, Inside
```

---

## 🧰 6. Key CLI Differences: Cisco IOS vs. Cisco ASA

| Operational Parameter | Cisco IOS Router | Cisco ASA Adaptive Security Appliance | Technical Rationale |
| :--- | :--- | :--- | :--- |
| **Interface Security Mapping** | Physical label only (`Gig0/0`) | Mandatory `nameif` and `security-level` | Establishes logical security zoning and policy enforcement |
| **Routing Table Command** | `show ip route` | `show route` | ASA CLI omits the `ip` keyword from routing commands |
| **Interface Status Command** | `show ip interface brief` | `show interface ip brief` | Word order differs in ASA parser |
| **Static Route Syntax** | `ip route <net> <mask> <next-hop>` | `route <nameif> <net> <mask> <next-hop>` | ASA requires explicit egress zone declaration |
| **OSPF Network Statement** | Wildcard mask (`0.0.0.255`) | Standard subnet mask (`255.255.255.0`) | ASA parser rejects inverse wildcard notation |
| **EIGRP Network Statement** | Optional wildcard mask | Explicit standard subnet mask | Standard subnet masks are strictly enforced on ASA |
| **Global Config Execution** | Requires `do` prefix inside config mode | Standard `show` commands run directly from config mode | Built-in CLI optimization on ASA appliances |

---

## 🚨 7. NOC Diagnostic & Troubleshooting Reference Table

| Symptom / Alert | Root Cause | Analytical Takeaway & Remediation |
| :--- | :--- | :--- |
| Interface remains down after IP assignment | Missing `nameif` or `security-level` definition | ASA interfaces require all 4 pillars (`nameif`, `security-level`, `ip address`, and `no shutdown`) before processing traffic. |
| Inside host pings WAN router, but receives timeout | ICMP is not inspected statefully by default | Reply packet drops at Security Level 0. Apply an inbound access list (`access-group <id> in interface Outside`) or enable ICMP in MPF inspection. |
| OSPF configuration rejected with `% Invalid input` | Entering inverse wildcard mask instead of subnet mask | Replace `0.0.0.255` with dotted-decimal subnet mask `255.255.255.0` in the `network` command. |
| OSPF routes not appearing in routing table | EIGRP running simultaneously with lower AD | EIGRP has an Administrative Distance of `90`, superseding OSPF's AD of `110`. Inspect routes with `show route eigrp` or `show ospf database`. |
| Cross-zone traffic between two DMZ interfaces drops | Same security level traffic blocked by default | When two interfaces share the same numeric security level (e.g., 50 to 50), traffic is denied unless `same-security-traffic permit inter-interface` is enabled. |

---

## 📋 8. Test Matrix Table

| Test ID | Test Scenario | Source Node | Destination Target | Protocol | Verification Metric | Status |
| :---: | :--- | :--- | :--- | :---: | :--- | :---: |
| **TC-01** | Point-to-Point Outside Link | `ASA` | `203.10.1.1` (`WAN-Router`) | ICMP | 100% reply rate (`!!!!!`) | ✅ **PASS** |
| **TC-02** | Point-to-Point Inside Link | `ASA` | `10.1.1.1` (`TestPC1`) | ICMP | 100% reply rate (`!!!!!`) | ✅ **PASS** |
| **TC-03** | Default Low-to-High Block | `WAN-Router` | `10.1.1.1` (`TestPC1`) | ICMP | Unsolicited ingress dropped at Level 0 | ✅ **PASS** |
| **TC-04** | Inbound ACL Transit Traversal | `TestPC1` | `203.10.1.1` (`WAN-Router`) | ICMP | Inbound ACL hit count increments; 0% loss | ✅ **PASS** |
| **TC-05** | Remote Loopback 0 Reachability | `TestPC1` | `22.2.2.2` (`WAN-Router`) | ICMP | Full transit traversal across ASA edge | ✅ **PASS** |
| **TC-06** | Remote Loopback 1 Reachability | `TestPC2` | `33.3.3.3` (`WAN-Router`) | ICMP | Full transit traversal across ASA edge | ✅ **PASS** |
| **TC-07** | OSPF Neighbor Adjacency | `ASA` | `203.10.1.1` | OSPF | Neighbor state reaches `FULL/BDR` | ✅ **PASS** |
| **TC-08** | EIGRP Neighbor Adjacency | `ASA` | `203.10.1.1` | EIGRP | Neighbor adjacency active with 0 Q Cnt | ✅ **PASS** |
| **TC-09** | AD Precedence Arbitration | `ASA` | `show route` | RIB | Prefix `22.2.2.0/24` installed via EIGRP `D` | ✅ **PASS** |

---

## 📦 9. Included Artifacts Checklist

* [x] **`cisco-asa-basic-initialization-routing.unl`** — Full EVE-NG topology XML configuration ready for immediate import.
* [x] **`topology.png`** — Clean high-definition visual topology diagram captured from EVE-NG.
* [x] **`asa-config.ios`** — Production Cisco ASA 5520 running configuration with security zones, ACLs, OSPF, and EIGRP.
* [x] **`wan-router-config.ios`** — Upstream Cisco IOS WAN Router running configuration with loopbacks, OSPF, and EIGRP.
* [x] **`wan-switch-config.ios`** — Layer 2 WAN transit switch access configuration.
* [x] **`lan-switch-config.ios`** — Layer 2 LAN distribution switch access configuration.
* [x] **`README.md`** — Comprehensive engineering documentation, CLI verification, and diagnostic guide.
