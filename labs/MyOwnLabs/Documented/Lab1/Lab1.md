# CCNA 200-301 — NorthStar Enterprise Network Challenge

> **Strict practical exam | 120 minutes | 100 points | Pass: 85/100**  
> **Answer key intentionally withheld until submission.**  
> Target: Cisco Packet Tracer 8.2.x. Check that your PT 3650 supports a routed Port-channel before beginning the timed attempt.

## A. Scenario

NorthStar Technologies needs a segmented, redundant office network. Build the network from scratch, implement routing and switching, and demonstrate failover. Your network must be **correct by protocol and path**, not merely pingable.

## B. Topology

### Inventory

| Name | Model | Role |
|---|---|---|
| DSW1, DSW2 | Cisco 3650-24PS | L3 distribution |
| ASW1, ASW2 | Cisco 2960-24TT | L2 access |
| TSW1 | Cisco 2960-24TT | Broadcast OSPF transit |
| R1, R2 | Cisco 2911 | Edge and backup routing |
| PC1–PC4 | PC-PT | User endpoints |

### Logical ASCII diagram

```text
                         DSW1 ===== L3 Po10 ===== DSW2
                          || \---- L2 trunk ----/ ||
                    LACP Po1                      PAgP Po2
                          ||                         ||
                         ASW1 ----- L2 trunk ------ ASW2
                         /  \                       /  \
                       PC1  PC2                   PC3  PC4

                 DSW1 ------ TSW1 ------- DSW2
                               |
                               R1
                               |
                          backup WAN
                               |
                               R2 ------- backup WAN ------- DSW2
```

The diagram is logical only; use this interface map for cabling.

### Physical interface map

| # | A | Port | B | Port | Purpose |
|---|---|---|---|---|---|
| 1 | DSW1 | Gi1/0/1 | DSW2 | Gi1/0/1 | Routed Po10 member |
| 2 | DSW1 | Gi1/0/2 | DSW2 | Gi1/0/2 | Routed Po10 member |
| 3 | DSW1 | Gi1/0/3 | DSW2 | Gi1/0/3 | Production L2 trunk |
| 4 | DSW1 | Gi1/0/4 | ASW1 | Fa0/1 | LACP Po1 member |
| 5 | DSW1 | Gi1/0/5 | ASW1 | Fa0/2 | LACP Po1 member |
| 6 | DSW2 | Gi1/0/4 | ASW2 | Fa0/1 | PAgP Po2 member |
| 7 | DSW2 | Gi1/0/5 | ASW2 | Fa0/2 | PAgP Po2 member |
| 8 | ASW1 | Fa0/24 | ASW2 | Fa0/24 | Production L2 trunk |
| 9 | DSW1 | Gi1/0/6 | TSW1 | Fa0/1 | OSPF broadcast LAN |
| 10 | DSW2 | Gi1/0/6 | TSW1 | Fa0/2 | OSPF broadcast LAN |
| 11 | R1 | Gi0/0 | TSW1 | Fa0/3 | OSPF broadcast LAN |
| 12 | R1 | Gi0/1 | R2 | Gi0/0 | Backup WAN |
| 13 | R2 | Gi0/1 | DSW2 | Gi1/0/8 | Backup WAN |
| 14 | PC1 | Fa0 | ASW1 | Fa0/10 | VLAN 10 |
| 15 | PC2 | Fa0 | ASW1 | Fa0/11 | VLAN 20 |
| 16 | PC3 | Fa0 | ASW2 | Fa0/10 | VLAN 30 |
| 17 | PC4 | Fa0 | ASW2 | Fa0/11 | VLAN 40 |

**Compatibility checkpoint (before clock starts):** Test whether the Packet Tracer 3650 supports `no switchport`, `channel-group`, and IPv4 addressing on `interface port-channel 10`. If it does not, document the limitation; do not silently replace L3 EtherChannel with a trunk.

## C. Addressing and VLAN requirements

**Major IPv4 block:** `172.16.40.0/22` — allocate all nine internal segments via VLSM yourself.

| Segment | VLAN | Minimum usable hosts |
|---|---:|---:|
| Engineering | 10 | 110 |
| Finance | 20 | 55 |
| Operations | 30 | 26 |
| Management department | 40 | 12 |
| Switch management | 99 | 5 |
| OSPF broadcast transit | 90 | 3 |
| Routed Po10 | — | 2 |
| R1–R2 WAN | — | 2 |
| R2–DSW2 WAN | — | 2 |

Allocate subnets in descending required host count starting at the lowest available address. Use the **smallest normal subnet** satisfying each requirement; no overlapping ranges and no `/31` or `/32` on point-to-point links.

| VLAN | Name | SVI/default gateway | Trunk rule |
|---|---|---|---|
| 10 | ENGINEERING | DSW1 | All production L2 trunks |
| 20 | FINANCE | DSW1 | All production L2 trunks |
| 30 | OPERATIONS | DSW2 | All production L2 trunks |
| 40 | MANAGEMENT | DSW2 | All production L2 trunks |
| 99 | NET-MGMT | DSW1 | All production L2 trunks |
| 90 | OSPF-TRANSIT | No SVI | TSW1 access ports only |
| 999 | NATIVE-BLACKHOLE | No SVI | Native VLAN on production trunks |

**Production trunks:** Native VLAN `999`, allowed VLANs **exactly** `10,20,30,40,99,999`. TSW1 ports to the routers/distribution switches are **access VLAN 90**, not trunks.

**Address positions:**

| Segment | Assignment convention |
|---|---|
| VLAN 10 | DSW1 first usable; PC1 last usable |
| VLAN 20 | DSW1 first usable; PC2 last usable |
| VLAN 30 | DSW2 first usable; PC3 last usable |
| VLAN 40 | DSW2 first usable; PC4 last usable |
| VLAN 99 | DSW1 first usable; ASW1 second; ASW2 third |
| OSPF LAN | DSW1 first usable; DSW2 second; R1 third |
| Po10 | DSW1 first usable; DSW2 second |
| R1–R2 | R1 first usable; R2 second |
| R2–DSW2 | R2 first usable; DSW2 second |

PC IPv4 IP/mask/gateway must be manually configured. No DHCP.

### Fixed external addresses and IPv6

| Device/interface | Address |
|---|---|
| R1 Loopback0 | `198.51.100.10/32` |
| R1 Loopback1 | `2001:DB8:100::1/64` |
| R2 Loopback0 | `2001:DB8:200::1/64` |
| R1 Gi0/1 | `2001:DB8:12::1/64` |
| R2 Gi0/0 | `2001:DB8:12::2/64` |

Enable IPv6 routing, add necessary IPv6 static routes, and demonstrate reachability to opposite IPv6 loopbacks.

## D. Scored tasks — 100 points

### D1. Addressing — 12 pts

- [ ] **6** — Calculate all nine IPv4 network addresses, prefixes, masks, ranges and broadcasts.
- [ ] **4** — Assign correct IPv4 IPs, masks and gateways across the topology.
- [ ] **2** — Configure IPv6 and prove IPv6 loopback reachability between R1/R2.

### D2. VLANs and routing — 12 pts

- [ ] **3** — Create exact VLAN numbers and names.
- [ ] **3** — Configure correct access VLANs and trunks.
- [ ] **2** — Set VLAN 999 native and exact permitted VLAN list on every production trunk.
- [ ] **4** — Configure five gateway/management SVIs on specified DSWs and enable L3 routing.

### D3. EtherChannel — 14 pts

- [ ] **4** — **Po1** DSW1 `active` ↔ ASW1 `passive`, LACP Layer 2 trunk.
- [ ] **4** — **Po2** DSW2 `desirable` ↔ ASW2 `auto`, PAgP Layer 2 trunk.
- [ ] **5** — **Po10** DSW1 ↔ DSW2, Layer 3 routed EtherChannel, static mode `on` both ends. IP on Po10, not members; no trunk.
- [ ] **1** — Show member ports bundled and demonstrate incompatible negotiation does not form a valid channel.

### D4. Rapid PVST+ — 12 pts

- [ ] **3** — DSW1 primary/root for VLANs **10, 20, 99**, base bridge priority **4096**.
- [ ] **3** — DSW2 primary/root for VLANs **30, 40**, base bridge priority **4096**.
- [ ] **2** — Opposite DSW secondary for each VLAN group, base priority **8192**.
- [ ] **1** — ASW1 Fa0/24 spanning-tree cost **30** for VLANs **10,20**.
- [ ] **3** — PortFast + BPDU Guard on PC-facing access ports only.

### D5. OSPFv2 — 19 pts

- [ ] **3** — Process 1, area 0; router IDs DSW1=`1.1.1.1`, DSW2=`2.2.2.2`, R1=`3.3.3.3`.
- [ ] **5** — Correct advertisements for VLANs, Po10, transit, R1 Loopback0; intended adjacencies reach FULL.
- [ ] **3** — Correct passive interfaces; no user VLAN neighbor relationships.
- [ ] **4** — On OSPF broadcast transit: DSW1 priority **200** and **DR**; DSW2 priority **100** and **BDR**; R1 priority **0** and **DROTHER**. Verify actual election.
- [ ] **4** — R1 is sole OSPF default originator; DSW1/DSW2 learn external OSPF default. R1's simulated static default points to `Null0`.

### D6. Static and floating routes — 13 pts

- [ ] **4** — Configure necessary backup-path static routing.
- [ ] **3** — Configure administrative distance **200** on specified floating routes.
- [ ] **4** — Fail R1 Gi0/0; PC1 still reaches R1 Loopback0 via DSW2→R2→R1.
- [ ] **2** — Restore R1 Gi0/0 and prove preferred OSPF routing returns.

Required routes:

1. DSW1 floating static `198.51.100.10/32` via **DSW2 Po10 IP**, AD 200.
2. DSW2 floating static `198.51.100.10/32` via **R2 directly connected WAN IP**, AD 200.
3. R2 static `198.51.100.10/32` via **R1 WAN IP**.
4. R2 static summary `172.16.40.0/22` via **DSW2 WAN IP**.
5. R1 floating static summary `172.16.40.0/22` via **R2 WAN IP**, AD 200.

The backup WAN links must remain up during failure injection. No OSPF on R2.

### D7. Hardening — 6 pts

- [ ] **1** — Correct hostnames.
- [ ] **2** — Unused switch access ports administratively shut down and assigned to VLAN 999.
- [ ] **2** — Working management SVIs and management default gateways on ASW1/ASW2.
- [ ] **1** — No PC-facing trunk ports.

### D8. Evidence — 12 pts

- [ ] **3** — VLAN/trunk/EtherChannel evidence.
- [ ] **2** — STP election and access protection evidence.
- [ ] **3** — OSPF neighbors, DR/BDR and routing evidence.
- [ ] **4** — Reachability, failover and recovery evidence.

**Total: 100 points.** D8 verification points are already included; the Section F checks below are not extra credit.

## E. Rules

1. No tutorials, copied configs, or external help during the timed attempt.
2. Do not change interface mapping, topology, VLAN IDs or fixed addresses.
3. Use correct VLSM with smallest valid prefixes and no overlap.
4. Use **SVIs**, not router-on-a-stick, for inter-VLAN routing.
5. Use exact EtherChannel modes and expected L2/L3 behavior.
6. R2 uses static routing only, **no OSPF**.
7. No OSPF neighbor formation on end-user VLANs.
8. Do not configure RIP, EIGRP, BGP or redistribution.
9. No alternative backup routes that bypass required failover.
10. No HSRP/VRRP/GLBP.
11. No PortFast/BPDU Guard on switch uplinks.
12. Pings alone do not prove correct route selection or channel negotiation.

## F. Verification checklist — 12 evidence points

**Run commands on the appropriate devices; interface-specific commands refer to the listed device and port.**

| Evidence | Commands | Expected | Pts |
|---|---|---|---:|
| VLAN/trunks | `show vlan brief`; `show interfaces trunk` | Correct membership, native 999, allowed list | 1 |
| EtherChannels | `show etherchannel summary`; `show interfaces port-channel 10`; `show ip interface brief` | Po1/Po2 are L2; Po10 routed; members bundled | 2 |
| STP roots | `show spanning-tree vlan 10`; `show spanning-tree vlan 30`; `show spanning-tree vlan 99` | Required primary/secondary bridge roles | 1 |
| STP safety | `show spanning-tree interface fa0/10 detail`; `show spanning-tree interface fa0/24 detail`; `show running-config` | Correct PortFast, BPDU Guard and port cost | 1 |
| OSPF adjacency/election | `show ip ospf neighbor`; `show ip ospf interface gigabitEthernet1/0/6`; `show ip ospf interface port-channel 10` | FULL neighbors; DSW1 DR, DSW2 BDR, R1 DROTHER; Po10 adjacency | 2 |
| OSPF routes/passive | `show ip route ospf`; `show ip protocols`; `show ip ospf interface brief` | Advertised VLAN routes, OSPF default, no user adjacencies | 1 |
| Baseline | `show ip route`; `ping 198.51.100.10`; `tracert 198.51.100.10` (on PCs) | Internal/external reachability using intended path | 1 |
| Failover | `show ip route 198.51.100.10`; `show ip ospf neighbor`; `ping 198.51.100.10`; `tracert 198.51.100.10` | Floating static selected after shutdown; loopback remains reachable | 2 |
| Recovery/IPv6 | `show ip route 198.51.100.10`; `show ipv6 interface brief`; `show ipv6 route` | OSPF preferred again, IPv6 loopback paths work | 1 |

### Baseline test matrix

| # | Source | Destination | Expected |
|---|---|---|---|
| 1 | PC1 | PC1 gateway | Success |
| 2 | PC1 | PC2 | Success |
| 3 | PC1 | PC3 | Success |
| 4 | PC1 | PC4 | Success |
| 5 | PC3 | PC2 | Success |
| 6 | ASW1 | ASW2 management IP | Success |
| 7 | DSW1 | DSW2 Po10 IP | Success |
| 8 | PC1 | 198.51.100.10 | Success |
| 9 | PC3 | 198.51.100.10 | Success |
| 10 | R1 | 2001:DB8:200::1 | Success |
| 11 | R2 | 2001:DB8:100::1 | Success |

1. Record DSW1/DSW2 routes to `198.51.100.10/32` **before** inducing failure.
2. Administratively shut **R1 Gi0/0 only**; wait for routing convergence.
3. Record DSW1/DSW2 routes and repeat tests **8 and 9**; capture traces.
4. Restore **R1 Gi0/0**; confirm preferred OSPF route returns; repeat tests.

## G. Hidden fault challenge

Five fault **categories** to diagnose:

- [ ] F1 — VLAN trunk inconsistency
- [ ] F2 — EtherChannel negotiation inconsistency
- [ ] F3 — OSPF adjacency/advertisement inconsistency
- [ ] F4 — Wrong route administrative distance / preference
- [ ] F5 — Wrong STP root election

**Important:** These are blind troubleshooting objectives, not actual preconfigured faults. A from-scratch topology cannot contain faults until someone injects them into the `.pkt` file. Your actual accidental mistakes count during grading. Locations are intentionally not provided.

## H. Grading

| Score | Outcome |
|---:|---|
| 95–100 | Excellent |
| 85–94 | Pass |
| 70–84 | Fail — near pass |
| 0–69 | Fail |

**Penalty guidelines (within each task's maximum):**

| Error | Deduction |
|---|---:|
| Wrong VLSM prefix / overlap | −2 per affected segment |
| Wrong interface IP/mask/gateway | −1 each |
| Missing VLAN/wrong access VLAN | −1 each |
| Native/allowed VLAN mismatch | −1 per logical trunk |
| Negotiation incompatibility | −2 per channel |
| Members not bundled | −1 per channel |
| Po10 built as L2 | −5 |
| Wrong STP root/secondary | −2 per VLAN group |
| Missing PortFast/BPDU Guard | −1 per affected switch |
| Missing OSPF neighbor | −2 per adjacency |
| Wrong DR/BDR | −2 |
| Missing OSPF prefix/default | −1 per requirement |
| Floating static incorrectly preferred while OSPF working | −2 |
| Backup path broken | −4 |
| Recovery unproven | −2 |
| Missing show evidence | Lose related D8 evidence points |

No double penalties for the same root cause within one task. **Hard cap: 69/100** if inter-VLAN routing fails entirely, all EtherChannels fail, or no OSPF adjacency is operational.

## I. ANSWER KEY - DO NOT READ UNTIL SUBMITTED

**LOCKED.** Complete configurations and the VLSM solution will be released after you submit your results. This README intentionally contains no solution.

---

## CCNA blueprint coverage (estimated; unofficial)

| Domain | Official weight | Approximate share of domain tested here |
|---|---:|---:|
| Network Fundamentals | 20% | 30–40% |
| Network Access | 20% | 40–50% |
| IP Connectivity | 25% | 75–85% |
| IP Services | 10% | 0–5% |
| Security Fundamentals | 15% | 5–10% |
| Automation and Programmability | 10% | 0% |

Overall, about **33–39% of the blueprint by breadth**, not by exam score prediction.

**Not tested:** TCP/UDP and extensive OSI fundamentals; wireless architecture; CDP/LLDP; HSRP/FHRP; NAT/PAT; DHCP/DNS/NTP/SNMP/syslog/QoS; SSH/AAA/ACLs; port security/DHCP snooping/DAI; VPN and wireless security; REST/JSON/SDN/Ansible/Terraform and automation; comprehensive IPv6.

## Strict schedule — 120 minutes

| Stage | Time |
|---|---:|
| Cabling and VLSM | 20 min |
| VLANs/SVIs/trunks/EtherChannels | 30 min |
| STP and OSPF | 30 min |
| Static routing and IPv6 | 15 min |
| Troubleshooting, evidence, failover | 25 min |

## Submission package

Submit the Packet Tracer `.pkt` file (if available), your full VLSM table, configs and the Section F outputs including screenshots/traces **before, during and after** the failure. Scoring will be point-by-point, strict, and without hints during the attempt.
