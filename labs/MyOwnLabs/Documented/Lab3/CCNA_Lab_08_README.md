# CCNA Lab 08 — Enterprise Layer 3 High Availability

**Platform:** Cisco Packet Tracer  
**Status:** In progress — routing backbone operational; HSRP redundancy and failover not yet validated  
**Provisional instructor grade:** **65/100** (based on submitted evidence, not a final functional certification)

## 1. Project overview

This enterprise-network lab combines VLSM, VLANs, 802.1Q trunks, inter-VLAN routing, a routed LACP EtherChannel, OSPFv2 area 0, HSRP gateway redundancy, and fault-tolerance testing.

### Devices

- 3 routers: R1, R2, R3
- 2 multilayer switches: DSW1, DSW2
- 2 access switches: ASW1, ASW2
- 4 PCs across VLANs 10, 20, 30 and 40

### Intended topology

```text
        R1 ---------------- R3 ---------------- R2
         |                   (loopback)          |
         |                                      |
        DSW1 ========== L3 Po1 ============== DSW2
         | \                                  / |
         |  \----- redundant L2 trunks -----/  |
         |                                     |
        ASW1 -------------------------------- ASW2
        /  \                                  /  \
      PC1  PC2                              PC3  PC4
```

**Diagram note:** The figure is a conceptual sketch, not an exact cabling diagram. Required Layer 2 edges are DSW1–ASW1, DSW2–ASW2, DSW1–ASW2, and DSW2–ASW1. There is **no requirement** for a direct ASW1–ASW2 link; the bottom line is illustrative only. The DSW1–DSW2 EtherChannel is **routed Layer 3**, not a VLAN-carrying trunk.

## 2. VLSM addressing plan

Base allocation: `172.20.0.0/21`.

| Segment | Network/prefix | Mask | Usable hosts | Broadcast |
|---|---|---|---|---|
| VLAN 10 — Engineering | `172.20.0.0/25` | `255.255.255.128` | `.1–.126` | `172.20.0.127` |
| VLAN 20 — Finance | `172.20.0.128/26` | `255.255.255.192` | `.129–.190` | `172.20.0.191` |
| VLAN 30 — Operations | `172.20.0.192/27` | `255.255.255.224` | `.193–.222` | `172.20.0.223` |
| VLAN 40 — Management | `172.20.0.224/28` | `255.255.255.240` | `.225–.238` | `172.20.0.239` |
| DSW1–DSW2 Po1 | `172.20.0.240/30` | `255.255.255.252` | `.241–.242` | `172.20.0.243` |
| DSW1–R1 | `172.20.0.244/30` | `255.255.255.252` | `.245–.246` | `172.20.0.247` |
| DSW2–R2 | `172.20.0.248/30` | `255.255.255.252` | `.249–.250` | `172.20.0.251` |
| R1–R3 | `172.20.0.252/30` | `255.255.255.252` | `.253–.254` | `172.20.0.255` |
| R2–R3 | `172.20.1.0/30` | `255.255.255.252` | `172.20.1.1–172.20.1.2` | `172.20.1.3` |

**All nine subnet calculations passed review (15/15).**

### VLAN SVI / HSRP IP plan

| VLAN | HSRP virtual gateway | DSW1 SVI | DSW2 SVI | Desired active |
|---|---|---|---|---|
| 10 | `172.20.0.1` | `172.20.0.2/25` | `172.20.0.3/25` | DSW1 |
| 20 | `172.20.0.129` | `172.20.0.130/26` | `172.20.0.131/26` | DSW1 |
| 30 | `172.20.0.193` | `172.20.0.195/27` | `172.20.0.194/27` | DSW2 |
| 40 | `172.20.0.225` | `172.20.0.227/28` | `172.20.0.226/28` | DSW2 |

**Note:** VLAN 30 and 40 interface assignments above reflect the actual submitted device outputs: DSW2 received the second usable IP and DSW1 the third. This differs from the original lab convention (DSW1 second, DSW2 third), but does not intrinsically break HSRP.

### Routed links — observed addresses

| Link | Side A | Side B |
|---|---|---|
| DSW1–DSW2 Po1 | DSW1 `172.20.0.241/30` | DSW2 `172.20.0.242/30` |
| DSW1–R1 | DSW1 `172.20.0.245/30` | R1 `172.20.0.246/30` |
| DSW2–R2 | DSW2 `172.20.0.249/30` | R2 `172.20.0.250/30` |
| R1–R3 | R1 `172.20.0.253/30` | R3 `172.20.0.254/30` |
| R2–R3 | R2 `172.20.1.2/30` | R3 `172.20.1.1/30` |

## 3. Switching and access layer

VLANs created on ASW1, ASW2, DSW1, and DSW2:

- **10:** Engineering
- **20:** Finance
- **30:** Operations
- **40:** Management
- **99:** Native VLAN

Observed host ports:

| Switch | Port | VLAN |
|---|---|---|
| ASW1 | Fa0/10 | 10 |
| ASW1 | Fa0/11 | 20 |
| ASW2 | Fa0/10 | 30 |
| ASW2 | Fa0/11 | 40 |

Observed trunks: ASW1 `Fa0/1`, ASW2 `Fa0/1`, DSW1 `Fa0/3`, DSW2 `Fa0/3`. All were shown trunking using 802.1Q, native VLAN 99, and allowed VLANs `10,20,30,40,99`.

**Outstanding:** The required cross-uplink trunks ASW1↔DSW2 and ASW2↔DSW1 have **not** been demonstrated. In submitted STP output, DSW1 and ASW2 each considered itself VLAN 10 root in its own L2 segment, supporting the diagnosis that VLAN domains were separated.

## 4. Layer 3 EtherChannel

Between DSW1 and DSW2:

- Port-channel: `Po1`
- Protocol: **LACP**
- Member ports: `Fa0/1` and `Fa0/2` on each distribution switch
- Type: **Layer 3 routed**
- Status: `Po1(RU)`, members `Fa0/1(P)` and `Fa0/2(P)`
- DSW1: `172.20.0.241/30`; DSW2: `172.20.0.242/30`
- OSPF adjacency across Po1: **FULL**

The supplied `show interfaces port-channel 1` reported **up/up** and 200,000 Kbit/s bandwidth. One-member failure survivability still requires a test.

## 5. OSPFv2 routing

- Process ID: **10**
- Area: **0**
- DSW1 router ID `1.1.1.1`
- DSW2 router ID `2.2.2.2`
- R1 router ID `3.3.3.3`
- R2 router ID `4.4.4.4`
- R3 router ID `5.5.5.5`

Confirmed FULL adjacencies:

- DSW1 ↔ DSW2 over Po1
- DSW1 ↔ R1
- DSW2 ↔ R2
- R1 ↔ R3
- R2 ↔ R3

Later outputs confirm that **both DSWs advertise all four VLAN prefixes** and list `Vlan10`, `Vlan20`, `Vlan30`, `Vlan40` as passive interfaces. R3 uses `Loopback1` (`203.0.113.3/32`), which is advertised in OSPF and learned on both DSWs as an `O` route. The lab originally requested `Loopback0`; the functional route is nevertheless present.

### Sample routing proof

```text
DSW1:
O 203.0.113.3 [110/3] via 172.20.0.246, GigabitEthernet0/1

DSW2:
O 203.0.113.3 [110/3] via 172.20.0.250, GigabitEthernet0/1

DSW1:
O 172.20.1.0/30 [110/3] via 172.20.0.246, GigabitEthernet0/1
                 [110/3] via 172.20.0.242, Port-channel1
```

This is evidence of remote-route learning and equal-cost multipath (ECMP) in the DSW1 routing table. It is **not** proof of completed failover testing.

## 6. HSRP configuration and diagnosed defect

Desired design:

| VLAN | DSW1 priority/role | DSW2 priority/role | Virtual IP |
|---|---|---|---|
| 10 | 120 / Active | 100 / Standby | `172.20.0.1` |
| 20 | 120 / Active | 100 / Standby | `172.20.0.129` |
| 30 | 100 / Standby | 120 / Active | `172.20.0.193` |
| 40 | 100 / Standby | 120 / Active | `172.20.0.225` |

Latest HSRP evidence showed **every group Active locally, with Standby `unknown`**. This **fails** functional FHRP redundancy.

Specific mismatch:

| VLAN | DSW1 HSRP version | DSW2 HSRP version |
|---|---|---|
| 10 | v2 | v1 |
| 20 | v2 | v1 |
| 30 | v1 | v2 |
| 40 | v1 | v2 |

**Cause 1:** HSRP protocol versions differ, so peers cannot form a common group.  
**Cause 2:** Missing Layer 2 cross-uplinks mean the required VLAN broadcast domains are not shared between both DSWs.

### Required repair

Set HSRP version 2 on the missing interfaces:

```cisco
! DSW1
conf t
interface vlan 30
 standby version 2
interface vlan 40
 standby version 2
end

! DSW2
conf t
interface vlan 10
 standby version 2
interface vlan 20
 standby version 2
end
```

Also build and verify the **cross-uplink L2 trunks**. Do not turn the routed EtherChannel into an L2 trunk. Verify STP and confirm no loops. After the repair, verify `show standby brief` reports expected Active/Standby neighbors rather than `unknown`.

## 7. Connectivity and failover test plan

| Test | Failure injected | Expected result | Evidence status |
|---|---|---|---|
| 1 | Disconnect one Po1 member | Po1 remains up | Not supplied |
| 2 | Disconnect DSW1–R1 | Traffic reroutes through DSW2 | Not supplied |
| 3 | Disconnect DSW2–R2 | Traffic reroutes through DSW1 | Not supplied |
| 4 | Shut down DSW1 | DSW2 takes over HSRP groups | Not supplied |
| 5 | Shut down DSW2 | DSW1 takes over HSRP groups | Not supplied |
| 6 | Disconnect R1–R3 | Traffic remains reachable via R2 | Not supplied |
| 7 | Disconnect R2–R3 | Traffic remains reachable via R1 | Not supplied |

An earlier PC-to-PC ping failed; no passing end-to-end ping evidence was subsequently submitted. Re-test **PC-to-PC**, **PC-to-gateway**, and **PC-to-203.0.113.3** after repairing Layer 2/HSRP.

## 8. Verification commands

```cisco
show vlan brief
show interfaces trunk
show spanning-tree vlan 10
show spanning-tree vlan 20
show spanning-tree vlan 30
show spanning-tree vlan 40
show etherchannel summary
show interfaces port-channel 1
show ip interface brief
show ip route
show ip ospf neighbor
show ip protocols
show standby brief
show standby
```

Run commands on the relevant devices; access switches do not need routed Port-channel interfaces.

## 9. Instructor assessment

| Category | Earned | Maximum | Comments |
|---|---:|---:|---|
| VLSM & subnetting | 15 | 15 | Correct allocation |
| VLANs & inter-VLAN routing | 12 | 15 | VLAN ports/trunks configured; missing L2 redundancy and ping proof |
| Layer 3 EtherChannel | 13 | 15 | LACP Po1 operational; member-failure test missing |
| OSPF | 17 | 20 | Backbone functioning; full host connectivity/failover not validated |
| HSRP | 8 | 20 | IPs/priorities correct; versions inconsistent, peers unknown |
| Failure tests | 0 | 15 | No test evidence submitted |
| **Total (provisional)** | **65** | **100** | **Incomplete** |

### Next actions

1. Make HSRP v2 consistent on both distribution switches.
2. Add the missing cross-uplink Layer 2 trunks; check native/allowed VLAN lists and STP.
3. Confirm correct HSRP Active/Standby roles on all VLANs.
4. Run host-to-host and remote-loopback pings.
5. Execute and document all seven failover cases.
6. Submit updated `show` commands and ping outputs for final grading.

---

**Important:** This README documents the **actual state demonstrated by the submitted command outputs**, including known defects and untested requirements. It should not be described as a fully completed or fault-tolerant deployment until the outstanding tests pass.
