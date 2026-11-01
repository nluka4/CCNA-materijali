# CCNA Lab #7 — Enterprise Switching & Routing

> **Topics:** VLANs, 802.1Q trunking, LACP EtherChannel, Rapid PVST+, HSRP, OSPFv2  
> **Platform:** Cisco Packet Tracer  
> **Status:** Configuration largely functional; final redundancy testing pending  
> **Provisional review:** 78/100 (evidence-based; not a final certification)

## 1. Objective

Build a redundant enterprise network with two distribution switches, two access switches, and three routers. Implement Layer 2 segmentation and redundancy, first-hop gateway redundancy, and OSPF routing. Validate normal connectivity and recovery after link failures.

**Scope:** VLANs, EtherChannel, STP, FHRP (HSRP), and OSPF only. No ACLs, NAT, IPv6, or DHCP.

## 2. Logical topology

```text
                     R3
                   /    \
                 R1      R2
                 |        |
               DSW1======DSW2
                | \      / |
                |  \    /  |
                |   \  /   |
                |    \/    |
                |    /\    |
                |   /  \   |
               ASW1      ASW2
              /    \     /    \
            PC1    PC2  PC3   SRV1
```

- `DSW1======DSW2`: two-member LACP **Layer 2** Port-Channel 1.
- **Each** access switch connects to **both** distribution switches using separate 802.1Q trunks.
- DSW1–R1, DSW2–R2, R1–R3, and R2–R3 are routed links.

### Devices

| Device | Function | Suggested model |
|---|---|---|
| DSW1 / DSW2 | Multilayer distribution | Catalyst 3560 |
| ASW1 / ASW2 | Access switches | Catalyst 2960 |
| R1 / R2 / R3 | Routing | Cisco 2911 |
| PC1 / PC2 / PC3 / SRV1 | End devices | Packet Tracer endpoints |

## 3. VLAN and IPv4 plan

Base allocation: `172.20.0.0/23`. VLSM uses the following ranges:

| VLAN | Name | Required hosts | Network | Mask | HSRP VIP | DSW1 SVI | DSW2 SVI |
|---|---|---:|---|---|---|---|---|
| 10 | ENGINEERING | 100 | `172.20.0.0/25` | `255.255.255.128` | `172.20.0.1` | `172.20.0.2` | `172.20.0.3` |
| 20 | SALES | 60 | `172.20.0.128/26` | `255.255.255.192` | `172.20.0.129` | `172.20.0.130` | `172.20.0.131` |
| 30 | MANAGEMENT | 28 | `172.20.0.192/27` | `255.255.255.224` | `172.20.0.193` | `172.20.0.194` | `172.20.0.195` |
| 40 | SERVERS | 12 | `172.20.0.224/28` | `255.255.255.240` | `172.20.0.225` | `172.20.0.226` | `172.20.0.227` |
| 99 | NATIVE | — | No user subnet | — | — | — | — |

Access port placement observed:

| Access switch | Interface | VLAN |
|---|---|---:|
| ASW1 | Fa0/10 | 10 |
| ASW1 | Fa0/11 | 20 |
| ASW2 | Fa0/10 | 30 |
| ASW2 | Fa0/11 | 40 |

All inter-switch trunks use native VLAN **99** and allow `10,20,30,40,99`.

### Routed transit links

The implementation shown in CLI uses the following links (each `/30`):

| Link | Network | Endpoint 1 | Endpoint 2 |
|---|---|---|---|
| DSW1–R1 | `10.0.0.0/30` | DSW1 `10.0.0.1` | R1 `10.0.0.2` |
| DSW2–R2 | `10.0.0.4/30` | DSW2 `10.0.0.5` | R2 `10.0.0.6` |
| R2–R3 | `10.0.0.8/30` | R2 `10.0.0.9` | R3 `10.0.0.10` |
| R1–R3 | `10.0.0.12/30` | R1 `10.0.0.13` | R3 `10.0.0.14` |

> Note: The original exercise specified a `10.10.0.0/24` transit pool; the implementation uses `10.0.0.0/24`. Addressing works, but it differs from the stated exercise requirement.

## 4. EtherChannel

**Port-Channel 1** connects DSW1 and DSW2 over Gi1/0/4 and Gi1/0/5.

| Setting | Observed / required |
|---|---|
| Protocol | LACP |
| Port-channel | `Po1(SU)` — Layer 2 and in use |
| Members | `Gi1/0/4(P)` and `Gi1/0/5(P)` on both DSWs |
| Encapsulation | 802.1Q |
| Native VLAN | 99 |
| Allowed VLANs | 10,20,30,40,99 |

**Verified:** Both member interfaces are bundled.  
**Still to verify:** Disconnect one member and confirm Po1 remains operational.

## 5. Spanning Tree Protocol

Target: **Rapid PVST+** on all switches, with root placement distributed between DSW1 and DSW2.

| VLAN | Expected root | Expected secondary | Evidence status |
|---|---|---|---|
| 10 | DSW1 | DSW2 | DSW1 root confirmed |
| 20 | DSW1 | DSW2 | Full root/secondary verification pending |
| 30 | DSW2 | DSW1 | DSW2 root confirmed |
| 40 | DSW2 | DSW1 | Full root/secondary verification pending |
| 99 | DSW1 | DSW2 | Full root/secondary verification pending |

Observed VLAN 30 access-switch port roles:

| Switch | Root port | Alternate blocking port |
|---|---|---|
| ASW1 | Fa0/2 | Fa0/1 |
| ASW2 | Fa0/1 | Fa0/2 |

**Outstanding discrepancy:** ASW2 still reports `Spanning tree enabled protocol ieee`, rather than `rstp`. Check Rapid PVST+ there. PortFast and BPDU Guard settings on host ports also require proof.

## 6. HSRP gateway redundancy

| VLAN | HSRP virtual IP | DSW1 | DSW2 |
|---|---|---|---|
| 10 | `172.20.0.1` | Active, priority 120 | Standby, priority 100 |
| 20 | `172.20.0.129` | Active, priority 120 | Standby, priority 100 |
| 30 | `172.20.0.193` | Standby, priority 100 | Active, priority 120 |
| 40 | `172.20.0.225` | Standby, priority 100 | Active, priority 120 |

**Verified:** Final `show standby brief` outputs show the intended active/standby roles and priorities; preemption is enabled on the active switches. The earlier unintended HSRP configuration on DSW1's router-facing Gi1/0/1 is no longer present.

**Still to verify:** Interface tracking and successful active-router handover when each active distribution switch loses its router-facing uplink.

## 7. OSPFv2

All internal routed adjacencies are in **area 0**, using process ID 1.

| Device | Lab-required router ID | Observed router ID |
|---|---|---|
| DSW1 | `1.1.1.1` | `1.1.1.1` |
| DSW2 | `2.2.2.2` | `2.2.2.2` |
| R1 | `3.3.3.3` | `10.0.0.2` |
| R2 | `4.4.4.4` | `10.0.0.6` |
| R3 | `5.5.5.5` | `203.0.113.1` |

**Verified:** The four expected router-to-router links establish FULL OSPF adjacencies. R3 learns all four VLAN networks. DSW1 and DSW2 learn `203.0.113.1/32` via OSPF.

R3 has `203.0.113.1/32` on **Loopback1**, not the originally requested Loopback0.

**Still to verify/correct:** Exact router IDs, passive-interface status on SVIs, specified preferred paths via OSPF cost, and alternate-path convergence after a failure. R3 currently installs equal-cost OSPF routes toward the VLAN subnets (ECMP), which is valid routing behavior but does not itself demonstrate the required preferred-path policy.

## 8. Connectivity tests

Based on screenshots submitted during the lab:

| Test destination | Result observed | Interpretation |
|---|---|---|
| `172.20.0.126` | 4/4 replies | Successful connectivity |
| `172.20.0.222` | First test 0/4, subsequent test successful | Reachability restored, cause of initial failure unconfirmed |
| `172.20.0.238` | 3/4 replies | Successful after initial timeout |

One initial ping timeout may reflect ARP resolution or Packet Tracer convergence, but should not be assumed harmless without repeated testing.

**Caveat:** Screenshots show connectivity, not complete failover validation.

## 9. Failure test matrix

- [ ] Disconnect ASW1–DSW1 and verify VLAN 10 reachability.
- [ ] Disconnect ASW2–DSW2 and verify VLAN 30 reachability.
- [ ] Disconnect one LACP member and verify Po1 remains up.
- [ ] Disconnect DSW1–R1 and verify expected HSRP tracking/failover.
- [ ] Disconnect DSW2–R2 and verify expected HSRP tracking/failover.
- [ ] Disconnect R1–R3 and verify alternate OSPF routing.
- [ ] Disconnect R2–R3 (after restoration) and verify alternate OSPF routing.
- [ ] Restore all links and verify expected stable roles and routes.

Document the before/after output and any dropped packets during convergence for each test.

## 10. Verification commands

```cisco
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree
show spanning-tree vlan 10
show spanning-tree vlan 30
show standby brief
show standby
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip interface brief
```

Layer 2 access switches are not expected to support routing/HSRP commands in this topology. Likewise, not every router supports EtherChannel or switch-style trunk verification.

## 11. Instructor review (provisional)

| Section | Score |
|---|---:|
| VLANs and VLSM | 15/15 |
| EtherChannel | 15/15 |
| STP / Rapid PVST+ | 15/20 |
| HSRP | 17/20 |
| OSPFv2 | 13/20 |
| Failure testing | 3/10 |
| **Provisional total** | **78/100** |

The score reflects the supplied outputs and earlier review. It is **not a final grade** because several behaviors have not been verified through failure tests.

### Remaining corrective actions

1. Check ASW2's Rapid PVST+ mode and confirm PortFast/BPDU Guard.
2. Match requested OSPF router IDs on R1, R2, and R3.
3. Confirm OSPF passive interfaces and route-cost preference.
4. Verify HSRP uplink tracking with physical failure tests.
5. Verify EtherChannel member failure and OSPF rerouting.
6. Capture final stable-state outputs after restoration.

## 12. Concept check

R3 has equal-cost paths to VLAN 10 through R1–DSW1 and R2–DSW2. **Can DSW2 route an arriving packet directly to a VLAN 10 host while DSW2 is the HSRP standby gateway?** Explain the difference between a router's ordinary Layer 3 forwarding and HSRP's first-hop gateway election.

---

*Documentation prepared from the provided Packet Tracer CLI outputs and ping screenshots. No unseen device configurations or failure outcomes have been assumed.*
