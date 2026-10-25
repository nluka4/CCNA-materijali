# CCNA Enterprise Network Lab — Hierarchical Redundancy

**Platform:** Cisco Packet Tracer  
**Scope:** VLANs, 802.1Q trunking, VLSM, Rapid PVST+, LACP EtherChannel, HSRP, OSPFv2, failover testing  
**Exam result:** **77/100** (strict practical assessment; passing threshold: 80/100)  
**Status:** Functional lab with documented specification deviations.

![Packet Tracer topology](assets/topology.png)

## Overview

This hands-on lab builds a small hierarchical enterprise network with **two access switches**, **two distribution switches**, and **three routers**. It tests segmented LAN design, redundant Layer 2 paths, first-hop gateway redundancy, dynamic routing and recovery from link or device failures.

The design uses Cisco 2960 Layer 2 switches and Cisco 2911 routers. Inter-VLAN routing runs via router-on-a-stick on R1 and R2; HSRP provides default gateway redundancy. R3 represents the remote network.

> **Note:** This README records the *submitted configuration and observed results*, not a corrected reference solution. See [Known issues](#known-issues-and-improvements).

## Topology and devices

| Role | Devices | Model |
|---|---|---|
| WAN/transit routers and default gateways | R1, R2, R3 (top router still named `Router` in submitted configuration) | 2911 |
| Distribution | DSW1, DSW2 | 2960 |
| Access | ASW1, ASW2 | 2960 |
| Clients | PC0, PC1, PC2, PC3 | PC-PT |
| Server | Server0 | Server-PT |

### Physical links

| Endpoint A | Port | Endpoint B | Port |
|---|---|---|---|
| R1 | G0/0 | DSW1 | Fa0/1 |
| R2 | G0/0 | DSW2 | Fa0/1 |
| R1 | G0/1 | R3 | G0/0 |
| R2 | G0/1 | R3 | G0/1 |
| R1 | G0/2 | R2 | G0/2 |
| DSW1 | Fa0/23 | DSW2 | Fa0/23 |
| DSW1 | Fa0/24 | DSW2 | Fa0/24 |
| ASW1 | Fa0/23 | DSW1 | Fa0/21 |
| ASW1 | Fa0/24 | DSW2 | Fa0/21 |
| ASW2 | Fa0/23 | DSW1 | Fa0/22 |
| ASW2 | Fa0/24 | DSW2 | Fa0/22 |
| PC0 | Fa0 | ASW1 | Fa0/2 |
| PC1 | Fa0 | ASW1 | Fa0/3 |
| PC2 | Fa0 | ASW2 | Fa0/2 |
| PC3 | Fa0 | ASW2 | Fa0/3 |
| Server0 | Fa0 | ASW2 | Fa0/4 |

## Addressing design

**Assigned LAN block:** `192.168.100.0/22`  
**VLSM allocations:**

| VLAN | Name | Required hosts | Network/prefix | Usable range | Broadcast | R1 | R2 | HSRP VIP |
|---|---|---:|---|---|---|---|---|---|
| 10 | ENGINEERING | 100 | `192.168.100.0/25` | `.1–.126` | `192.168.100.127` | `.1` | `.2` | `.3` |
| 20 | SALES | 50 | `192.168.100.128/26` | `.129–.190` | `192.168.100.191` | `.129` | `.130` | `.131` |
| 30 | MANAGEMENT | 25 | `192.168.100.192/27` | `.193–.222` | `192.168.100.223` | `.193` | `.194` | `.195` |
| 40 | SERVERS | 12 | `192.168.100.224/28` | `.225–.238` | `192.168.100.239` | `.225` | `.226` | `.227` |
| 99 | NATIVE | — | Not routed | — | — | — | — | — |

All shortened addresses in the last five columns use prefix `192.168.100`. The remaining addresses in the parent `/22` were not allocated.

**Original endpoint allocation rule:** fourth usable address for each PC; fifth usable address in VLAN 40 for Server0. Full endpoint network settings were not independently captured in the submitted text, although connectivity was demonstrated.

### Router transit networks

| Link | Network | Lower-numbered endpoint | Higher-numbered endpoint |
|---|---|---|---|
| R1 ↔ R2 | `10.0.12.0/30` | R1 `10.0.12.1` | R2 `10.0.12.2` |
| R1 ↔ R3 | `10.0.13.0/30` | R1 `10.0.13.1` | R3 `10.0.13.2` |
| R2 ↔ R3 | `10.0.23.0/30` | R2 `10.0.23.1` | R3 `10.0.23.2` |

**R3 loopback:** `203.0.113.1/32`, used as a remote routing and connectivity target.

## Configuration objectives

### 1. VLANs and 802.1Q trunks

- Create VLANs `10`, `20`, `30`, `40`, `99` on all four switches.
- Place PC0 in VLAN 10, PC1 in VLAN 20, PC2 in VLAN 30, PC3 and Server0 in VLAN 40.
- Configure all router/distribution/access uplinks as 802.1Q trunks where applicable.
- Use **native VLAN 99** and allowed VLANs `10,20,30,40,99`.
- Configure router-on-a-stick subinterfaces on R1 and R2 for VLANs 10/20/30/40, plus the native VLAN 99 subinterface without an IP address.
- Use PortFast and BPDU Guard on host-facing access ports.

**Submitted outcome:** VLAN assignment, trunking, allowed VLANs and native VLAN verified in switch outputs. Unused ports were not shut down and `switchport nonegotiate` was absent from distribution router-facing trunks.

### 2. LACP EtherChannel

Bundle DSW1/DSW2 `Fa0/23` and `Fa0/24` into `Port-channel1` using **LACP active/active**. Configure Po1 as a trunk with native VLAN 99 and the approved VLAN list.

**Submitted outcome:** Matching LACP configuration and an active Po1 trunk; member-link failure test maintained connectivity. A clean `show etherchannel summary` output from DSW1/DSW2 was not captured because `how etherchannel summary` was entered in the pasted terminal transcript.

### 3. STP root bridge placement

| VLAN | Primary root | Primary priority | Secondary | Secondary priority |
|---|---|---:|---|---:|
| 10 | DSW1 | 24576 | DSW2 | 28672 |
| 20 | DSW1 | 24576 | DSW2 | 28672 |
| 30 | DSW2 | 24576 | DSW1 | 28672 |
| 40 | DSW2 | 24576 | DSW1 | 28672 |
| 99 | DSW1 | 24576 | DSW2 | 28672 |

**Required mode:** Rapid PVST+ across all four switches.  
**Submitted outcome:** Correct root placement for verified VLANs; ASW1/ASW2 use Rapid PVST+, but DSW1/DSW2 use legacy PVST+ (`spanning-tree mode pvst`). Access-switch forwarding and alternate ports were visible in the STP output.

### 4. HSRP gateway redundancy

| Group/VLAN | Expected active | Expected standby | Virtual IP | Active priority | Standby priority |
|---|---|---|---|---:|---:|
| 10 | R1 | R2 | `192.168.100.3` | 120 | 100 |
| 20 | R1 | R2 | `192.168.100.131` | 120 | 100 |
| 30 | R2 | R1 | `192.168.100.195` | 120 | 100 |
| 40 | R2 | R1 | `192.168.100.227` | 120 | 100 |

**Required:** HSRP version 2, preemption on both routers, automatic failover and reversion after recovery.  
**Submitted outcome:** Correct initial Active/Standby roles and virtual IPs. HSRP **version 2 and preemption were not configured**, so the complete recovery requirement was not met.

### 5. OSPFv2 area 0

- Process ID **10** on all routers.
- Router IDs: R1 `1.1.1.1`, R2 `2.2.2.2`, R3 `3.3.3.3`.
- Advertise all three transit `/30` networks, the four VLAN networks from R1/R2, and R3's loopback `/32`.
- Use passive interfaces for user VLAN subinterfaces (advertise networks without forming LAN-side adjacencies).
- Verify remote loopback connectivity without static-route substitutes.

**Submitted outcome:** All transit adjacencies formed and R3 learned VLAN networks by OSPF. R1 and R2 also formed four *unnecessary* OSPF adjacencies over VLAN 10/20/30/40 because only the physical `G0/0` parent was marked passive, not the VLAN subinterfaces.

## Validation evidence

| Scenario | Evidence/result | Assessment |
|---|---|---|
| ASW1–DSW1 uplink failure | Host ping responses resumed after link failure | Passed |
| ASW2–DSW2 uplink failure | Host ping responses resumed after link failure | Passed |
| EtherChannel member failure | Traffic continued on remaining inter-distribution path | Passed |
| R1 HSRP failure | State-change messages and connectivity evidence | Partially demonstrated |
| R1 recovery/reversion | No preemption configured | Not satisfied |
| R2–R3 link failure | Remote loopback still responded | Reachability shown; exact alternate next hop not captured |
| End-to-end remote reachability | Successful pings to `203.0.113.1` | Passed |

Short initial ping losses can reflect ARP learning or protocol convergence; a single timeout alone is not proof of protocol failure.

### Useful verification commands

Run these on the **relevant device type**:

```text
# Layer 2 switches
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree vlan 10
show spanning-tree vlan 30

# R1 and R2
show standby brief
show ip ospf neighbor
show ip route

# R3
show ip ospf neighbor
show ip route
```

Do not expect router-only `show standby` / `show ip ospf neighbor` on 2960 access switches.

## Assessment: 77/100

| Category | Maximum | Awarded |
|---|---:|---:|
| VLSM / IP addressing | 15 | 14 |
| VLANs / trunking | 15 | 13 |
| EtherChannel | 10 | 10 |
| STP | 15 | 10 |
| HSRP | 20 | 12 |
| OSPF | 15 | 11 |
| Failover testing | 10 | 7 |
| **Total** | **100** | **77** |

Assessment used an **80/100** pass threshold and strict conformance to the given specification. The lab achieved useful end-to-end connectivity and multiple redundancy tests, but some required protocol details were incomplete.

## Known issues and improvements

- [ ] Change DSW1/DSW2 to Rapid PVST+ (currently PVST+).
- [ ] Configure HSRP version 2 consistently on R1/R2.
- [ ] Enable HSRP preemption and verify takeover **and** original-role restoration.
- [ ] Make VLAN subinterfaces passive under OSPF while preserving transit-neighbor adjacencies.
- [ ] Shut down all genuinely unused switch interfaces.
- [ ] Disable DTP on the DSW router-facing trunks.
- [ ] Rename remote `Router` hostname to `R3`.
- [ ] Capture complete per-endpoint IP/mask/gateway values and all expected validation outputs.

## Repository contents

```text
ccna-enterprise-lab/
├── README.md                # Lab specification, observations and assessment
├── configurations.txt       # Exported running configurations (as submitted)
└── assets/
    └── topology.png        # Original topology screenshot
```

## Learning outcomes

This exercise demonstrates practical knowledge of hierarchical campus switching, VLAN segmentation, first-hop gateway redundancy, dynamic route exchange, and fault testing. It also highlights a key distinction in enterprise networking: **functional connectivity does not automatically imply full compliance with design and redundancy requirements.**
