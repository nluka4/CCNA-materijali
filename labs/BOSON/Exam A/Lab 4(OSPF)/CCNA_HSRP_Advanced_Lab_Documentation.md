# CCNA Advanced HSRP Lab — Configuration, Tests & Review

**Lab:** Multi-VLAN HSRP Gateway Redundancy, Load Sharing, Tracking, and Recovery  
**Simulator:** Cisco Packet Tracer (3560 distribution, 2960 access, 2911 routers)  
**Recorded grade:** **79/100 (B)** — based on submitted screenshots, command output, and theory answers.  
**Status:** Base HSRP election and load-sharing verified; HSRP v2 and tracking-driven Active-role migration still require correction/retest.

> **Scope:** VLAN, STP, trunks, EtherChannel, SVIs, and OSPF were *preconfigured prerequisites*, not the student's graded lab work. The port mapping below is the **actual topology verified using `show cdp neighbors`**, and supersedes the first draft's incorrect interface assignments. Configurations are reconstructed reference baseline, not an export of the final `.pkt` device configurations. Paste sections into the correct devices and verify on your Packet Tracer IOS.

## 1. Topology and cabling (verified)

```text
                            R3 (Lo0: 203.0.113.1/32)
                           /                       \
                         R1                         R2
                          |                          |
                        DSW1 ======= Po1 ======== DSW2
                         | \                       / |
                         |  \                     /  |
                         |   ASW1           ASW2     |
                         |   /  \           /   \     |
                         +--/    PC2      PC3  SRV1---+
                            PC1
```

| Device A | Port | Device B | Port | Purpose |
|---|---|---|---|---|
| R3 | Gi0/0 | R1 | Gi0/1 | Routed |
| R3 | Gi0/1 | R2 | Gi0/1 | Routed |
| R1 | Gi0/0 | DSW1 | Gi0/1 | Routed uplink |
| R2 | Gi0/0 | DSW2 | Gi0/1 | Routed uplink |
| DSW1 | Fa0/1 | DSW2 | Fa0/1 | EtherChannel Po1 member |
| DSW1 | Fa0/2 | DSW2 | Fa0/2 | EtherChannel Po1 member |
| DSW1 | Fa0/3 | ASW1 | Fa0/1 | Access switch trunk |
| DSW1 | Fa0/4 | ASW2 | Fa0/2 | Access switch trunk |
| DSW2 | Fa0/4 | ASW1 | Fa0/2 | Access switch trunk |
| DSW2 | Fa0/3 | ASW2 | Fa0/1 | Access switch trunk |
| ASW1 | Fa0/10 | PC1 | Fa0 | VLAN 10 |
| ASW1 | Fa0/11 | PC2 | Fa0 | VLAN 20 |
| ASW2 | Fa0/10 | PC3 | Fa0 | VLAN 30 |
| ASW2 | Fa0/11 | SRV1 | Fa0 | VLAN 40 |

> On the 3560, `switchport trunk encapsulation dot1q` must precede `switchport mode trunk`, otherwise IOS may reject trunk mode because encapsulation is `Auto`.

## 2. Addressing plan

| VLAN | Name | Subnet | DSW1 SVI | DSW2 SVI | HSRP VIP | Normal Active |
|---|---|---|---|---|---|---|
| 10 | ENGINEERING | 10.10.10.0/24 | 10.10.10.2 | 10.10.10.3 | 10.10.10.1 | DSW1 |
| 20 | SALES | 10.20.20.0/24 | 10.20.20.2 | 10.20.20.3 | 10.20.20.1 | DSW2 |
| 30 | MANAGEMENT | 10.30.30.0/24 | 10.30.30.2 | 10.30.30.3 | 10.30.30.1 | DSW1 |
| 40 | SERVERS | 10.40.40.0/24 | 10.40.40.2 | 10.40.40.3 | 10.40.40.1 | DSW2 |
| 200 | TRANSIT | 10.255.200.0/30 | 10.255.200.1 | 10.255.200.2 | None | Routed DSW-to-DSW adjacency |

| Routed segment | Endpoint A | Endpoint B |
|---|---|---|
| DSW1–R1, 10.255.1.0/30 | DSW1 10.255.1.1 | R1 10.255.1.2 |
| DSW2–R2, 10.255.2.0/30 | DSW2 10.255.2.1 | R2 10.255.2.2 |
| R1–R3, 10.255.13.0/30 | R1 10.255.13.1 | R3 10.255.13.2 |
| R2–R3, 10.255.23.0/30 | R2 10.255.23.1 | R3 10.255.23.2 |
| R3 loopback | 203.0.113.1/32 | Test target |

| End host | IP/mask | Gateway |
|---|---|---|
| PC1 | 10.10.10.10/24 | 10.10.10.1 |
| PC2 | 10.20.20.10/24 | 10.20.20.1 |
| PC3 | 10.30.30.10/24 | 10.30.30.1 |
| SRV1 | 10.40.40.10/24 | 10.40.40.1 |

## 3. Baseline configurations (no HSRP — setup prerequisites)

### DSW1 (3560)

```cisco
conf t
hostname DSW1
ip routing
vlan 10
 name ENGINEERING
vlan 20
 name SALES
vlan 30
 name MANAGEMENT
vlan 40
 name SERVERS
vlan 200
 name TRANSIT
!
interface vlan 10
 ip address 10.10.10.2 255.255.255.0
 no shutdown
interface vlan 20
 ip address 10.20.20.2 255.255.255.0
 no shutdown
interface vlan 30
 ip address 10.30.30.2 255.255.255.0
 no shutdown
interface vlan 40
 ip address 10.40.40.2 255.255.255.0
 no shutdown
interface vlan 200
 ip address 10.255.200.1 255.255.255.252
 no shutdown
!
interface gigabitEthernet 0/1
 no switchport
 ip address 10.255.1.1 255.255.255.252
 no shutdown
!
interface range fastEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,200
 channel-group 1 mode active
 no shutdown
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,200
 no shutdown
!
interface range fastEthernet 0/3 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,30 root primary
spanning-tree vlan 20,40 root secondary
spanning-tree vlan 200 root primary
!
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface gigabitEthernet 0/1
 no passive-interface vlan 200
 network 10.10.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0
 network 10.30.30.0 0.0.0.255 area 0
 network 10.40.40.0 0.0.0.255 area 0
 network 10.255.1.0 0.0.0.3 area 0
 network 10.255.200.0 0.0.0.3 area 0
end
write memory
```

### DSW2 (3560)

```cisco
conf t
hostname DSW2
ip routing
vlan 10
 name ENGINEERING
vlan 20
 name SALES
vlan 30
 name MANAGEMENT
vlan 40
 name SERVERS
vlan 200
 name TRANSIT
!
interface vlan 10
 ip address 10.10.10.3 255.255.255.0
 no shutdown
interface vlan 20
 ip address 10.20.20.3 255.255.255.0
 no shutdown
interface vlan 30
 ip address 10.30.30.3 255.255.255.0
 no shutdown
interface vlan 40
 ip address 10.40.40.3 255.255.255.0
 no shutdown
interface vlan 200
 ip address 10.255.200.2 255.255.255.252
 no shutdown
!
interface gigabitEthernet 0/1
 no switchport
 ip address 10.255.2.1 255.255.255.252
 no shutdown
!
interface range fastEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,200
 channel-group 1 mode active
 no shutdown
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,200
 no shutdown
!
interface range fastEthernet 0/3 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
!
spanning-tree mode rapid-pvst
spanning-tree vlan 20,40 root primary
spanning-tree vlan 10,30 root secondary
spanning-tree vlan 200 root secondary
!
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface gigabitEthernet 0/1
 no passive-interface vlan 200
 network 10.10.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0
 network 10.30.30.0 0.0.0.255 area 0
 network 10.40.40.0 0.0.0.255 area 0
 network 10.255.2.0 0.0.0.3 area 0
 network 10.255.200.0 0.0.0.3 area 0
end
write memory
```

### ASW1 (2960)

```cisco
conf t
hostname ASW1
vlan 10
 name ENGINEERING
vlan 20
 name SALES
vlan 30
 name MANAGEMENT
vlan 40
 name SERVERS
!
interface fastEthernet 0/10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
interface fastEthernet 0/11
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
interface range fastEthernet 0/1 - 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
spanning-tree mode rapid-pvst
end
write memory
```

### ASW2 (2960)

```cisco
conf t
hostname ASW2
vlan 10
 name ENGINEERING
vlan 20
 name SALES
vlan 30
 name MANAGEMENT
vlan 40
 name SERVERS
!
interface fastEthernet 0/10
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
interface fastEthernet 0/11
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
interface range fastEthernet 0/1 - 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
spanning-tree mode rapid-pvst
end
write memory
```

### R1 (2911)

```cisco
conf t
hostname R1
interface gigabitEthernet 0/0
 ip address 10.255.1.2 255.255.255.252
 no shutdown
interface gigabitEthernet 0/1
 ip address 10.255.13.1 255.255.255.252
 no shutdown
router ospf 1
 router-id 11.11.11.11
 network 10.255.1.0 0.0.0.3 area 0
 network 10.255.13.0 0.0.0.3 area 0
end
write memory
```

### R2 (2911)

```cisco
conf t
hostname R2
interface gigabitEthernet 0/0
 ip address 10.255.2.2 255.255.255.252
 no shutdown
interface gigabitEthernet 0/1
 ip address 10.255.23.1 255.255.255.252
 no shutdown
router ospf 1
 router-id 22.22.22.22
 network 10.255.2.0 0.0.0.3 area 0
 network 10.255.23.0 0.0.0.3 area 0
end
write memory
```

### R3 (2911)

```cisco
conf t
hostname R3
interface gigabitEthernet 0/0
 ip address 10.255.13.2 255.255.255.252
 no shutdown
interface gigabitEthernet 0/1
 ip address 10.255.23.2 255.255.255.252
 no shutdown
interface loopback 0
 ip address 203.0.113.1 255.255.255.255
router ospf 1
 router-id 33.33.33.33
 network 10.255.13.0 0.0.0.3 area 0
 network 10.255.23.0 0.0.0.3 area 0
 network 203.0.113.1 0.0.0.0 area 0
end
write memory
```

> **Baseline note:** If adapting an already configured Packet Tracer file, remove obsolete `channel-group 1` assignments from Fa0/23–24 first. Both ends of Po1 must use **Fa0/1–2**, with identical trunk encapsulation, allowed VLANs, and LACP mode. Expect `Po1(SU)`, members marked `(P)`.

## 4. HSRP task specifications (graded exercise)

- Enable **HSRP version 2** for groups 10, 20, 30, and 40 on both DSWs.
- Use VLAN's `.1` virtual IP.
- Normal preference: DSW1 Active for VLANs 10/30; DSW2 Active for VLANs 20/40.
- Preferred Active priority: **120**; peer priority: **100** (one valid design).
- Enable preemption on routers that must reclaim the Active role.
- Track the **local routed uplink Gi0/1** on each DSW, decrementing by enough that the other switch's priority becomes **strictly higher**.
- Verify both local-uplink failure and automatic restoration without manually restarting HSRP.

### Reference configuration — HSRP v2 and tracking correction

This is an **improved solution example** for studying after attempting the lab; it was **not** the exact originally submitted configuration. On this topology, a decrement of **30** produces 120 → 90, which falls below the peer's 100. The other switch must be allowed to preempt to make a *live*, reachable Active router relinquish its role due to tracking.

**DSW1:**

```cisco
conf t
interface vlan 10
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 120
 standby 10 preempt
 standby 10 track gigabitEthernet 0/1 30
interface vlan 20
 standby version 2
 standby 20 ip 10.20.20.1
 standby 20 priority 100
 standby 20 preempt
interface vlan 30
 standby version 2
 standby 30 ip 10.30.30.1
 standby 30 priority 120
 standby 30 preempt
 standby 30 track gigabitEthernet 0/1 30
interface vlan 40
 standby version 2
 standby 40 ip 10.40.40.1
 standby 40 priority 100
 standby 40 preempt
end
write memory
```

**DSW2:**

```cisco
conf t
interface vlan 10
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 100
 standby 10 preempt
interface vlan 20
 standby version 2
 standby 20 ip 10.20.20.1
 standby 20 priority 120
 standby 20 preempt
 standby 20 track gigabitEthernet 0/1 30
interface vlan 30
 standby version 2
 standby 30 ip 10.30.30.1
 standby 30 priority 100
 standby 30 preempt
interface vlan 40
 standby version 2
 standby 40 ip 10.40.40.1
 standby 40 priority 120
 standby 40 preempt
 standby 40 track gigabitEthernet 0/1 30
end
write memory
```

> **Device support:** HSRP v2 and interface-tracking syntax vary by simulated IOS. If a command is rejected, use `standby ?`, `standby <group> ?`, and check Packet Tracer device capabilities. Interface tracking only detects that local port going down; if the far-side or upstream network fails while the port remains up, additional tracking mechanisms may be required.

## 5. Verification checklist

**Base network:**

```cisco
show cdp neighbors
show etherchannel summary
show interfaces trunk
show spanning-tree vlan 10
show ip interface brief
show ip ospf neighbor
show ip route
ping 203.0.113.1
```

**HSRP:**

```cisco
show standby brief
show standby
show running-config
```

Expected normal state:

| VLAN | DSW1 state | DSW2 state |
|---|---|---|
| 10 | Active | Standby |
| 20 | Standby | Active |
| 30 | Active | Standby |
| 40 | Standby | Active |

For HSRP v2, verify output indicates version 2, not `v1 default` MAC addresses. Cisco HSRP v1 group 10 uses `0000.0c07.ac0a`; HSRP v2 group 10 uses `0000.0c9f.f00a`.

**Before treating a failure as proven:** Collect state **before failure**, **after convergence**, and **after link recovery**. Run repeating pings where possible and separate transient ARP/STP/HSRP reconvergence loss from sustained packet loss.

## 6. Six failure tests

| Test | Action | Required behavior |
|---|---|---|
| 1 | Fully disable DSW1 | DSW2 becomes Active for all four VLANs; clients regain connectivity |
| 2 | Fully disable DSW2 | DSW1 becomes Active for all four VLANs; clients regain connectivity |
| 3 | Disable **DSW1 Gi0/1** toward R1 only | DSW2 becomes Active for VLANs 10/30 via priority tracking; downstream clients can still reach test loopback |
| 4 | Disable **DSW2 Gi0/1** toward R2 only | DSW1 becomes Active for VLANs 20/40 via priority tracking; clients can still reach test loopback |
| 5 | DSW1 uplink down, DSW2 uplink up | All four HSRP groups on DSW2 after convergence; normal load sharing returns when uplink restored |
| 6 | DSW1 uplink up, DSW2 uplink down | All four HSRP groups on DSW1 after convergence; normal load sharing returns when uplink restored |

**Important nuance:** In this lab, OSPF on VLAN 200 gives the distribution switches an alternate route through each other. Therefore Internet/test-loopback reachability alone does **not** prove tracking-based HSRP migration. Verify the **Active** and **Standby** columns.

### Submission evidence

- `show standby brief` from both switches in the normal state.
- `show standby brief` from both switches after each upstream-link failure and again after recovery.
- `show standby` to confirm version, tracking effects, priority and preemption.
- `show etherchannel summary`, `show interfaces trunk`, `show ip ospf neighbor` to validate the prerequisites.
- Ping results from PC1, PC2, PC3, and SRV1 to `203.0.113.1`.

## 7. Twelve theory answers — connected reasoning

1. **What determines Active?** Normally higher **HSRP priority** wins the election, with preemption determining whether a higher-priority newcomer can displace an already-Active router. Default priority is 100.
2. **Tie-breaker?** Highest **interface IP address** among the competing HSRP routers, not the OSPF router ID.
3. **Why preempt?** After the preferred gateway returns, it can **reclaim Active** from the temporary replacement. Preemption is also essential when tracking lowers the priority of a still-alive Active device and the peer needs to take over.
4. **Why can an Active gateway black-hole traffic?** Its SVI and HSRP heartbeat can remain alive even if its only useful upstream route fails. It keeps receiving traffic but cannot forward it. This particular lab's OSPF path via VLAN 200 can prevent that black hole without changing the HSRP role.
5. **What does tracking change?** Failure of a monitored interface **reduces HSRP priority**, permitting a better-connected peer to become Active when the peer is configured to preempt.
6. **120 − 15?** **105**.
7. **120 vs. 110; decrement 5?** No: 120 − 5 = **115**, still above 110. To guarantee a strict priority advantage for the peer, decrement by more than 10.
8. **Does the PC know which physical switch is Active?** Normally no. It sends to the same gateway **VIP and virtual MAC** regardless of the current owner.
9. **What IP does the PC ARP for?** The **HSRP virtual IP** (for example, `10.10.10.1`), because that is its configured gateway—not either physical SVI IP.
10. **What happens to the VIP on Active failure?** The **VIP remains the same**; the new Active takes ownership. No PC default gateway update is needed.
11. **What happens to the virtual MAC?** The **same virtual MAC** is now sourced by the new Active. Switches relearn its port location; hosts' gateway ARP entries generally remain valid.
12. **Why load-share across VLANs but not within one group?** Each group elects **one Active**. Multiple VLANs have independent HSRP groups, so VLAN 10/30 may use DSW1 while VLAN 20/40 uses DSW2. Two Active nodes in one group would compete for the same gateway identity.

**One-line memory aid:** **VIP/MAC give clients a stable gateway → priority chooses the preferred owner → standby takes over upon gateway loss → tracking detects selected uplink failures → preempt permits a better-priority peer to take over and later restores the intended design.**

## 8. Review and final grade

**Recorded grade: 79/100 (B)**. This is the grade previously assigned to the screenshots and outputs; it is **not** a new live re-evaluation of the Packet Tracer file.

| Area | Points |
|---|---:|
| VLAN/SVI addressing | 10/10 |
| HSRP VIPs/groups | 10/10 |
| Active/Standby design | 15/15 |
| Priorities | 10/10 |
| Preemption | 10/10 |
| Interface tracking | 8/20 |
| Failure recovery | 11/15 |
| Verification | 3/5 |
| Theory | 2/5 |
| **Total** | **79/100** |

### Evidence-based observations

- **Verified:** normal group ownership matched the design: DSW1 Active for 10/30 and DSW2 Active for 20/40; reciprocal Standby peers appeared in `show standby brief`.
- **Verified:** routed reachability to `203.0.113.1` eventually succeeded in supplied ping captures; transient losses occurred during some tests.
- **Not met:** output showed `v1 default` virtual MAC, so HSRP **v2** was not configured.
- **Not met:** after a DSW2 tracked uplink failure, priority dropped from 120 to 110 while DSW1 remained at 100. DSW2 thus remained Active for VLANs 20/40; the required migration did **not** happen.
- **Not independently demonstrated:** complete evidence for tests 5 and 6 and measurement of convergence/outage time.
- **Theory correction:** hosts ARP for the **virtual IP**, the virtual MAC moves with Active ownership, and the group must have **one Active**.

### Recommended retest to reach 90+

- [ ] Configure `standby version 2` on both switches for every user VLAN.
- [ ] Ensure the tracking decrement lowers preferred Active priority **below** its peer.
- [ ] Ensure the peer is able to preempt even when the existing Active is still reachable but has lost priority.
- [ ] Re-run upstream failures in both directions and prove role takeover from `show standby brief`.
- [ ] Restore upstream links and prove the original load-sharing split returns.
- [ ] Explain VIP versus physical SVI, virtual MAC ownership, and black-hole prevention without notes.

---

*Documentation compiled from the lab design and the user's submitted CLI outputs/screenshots. It contains a corrected baseline rather than claiming the original mis-mapped baseline was valid.*
