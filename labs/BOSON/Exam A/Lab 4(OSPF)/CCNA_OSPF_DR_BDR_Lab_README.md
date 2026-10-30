# CCNA Practical Lab — OSPF DR/BDR Election

**Platform:** Cisco Packet Tracer  
**Devices:** 3 × Cisco 2811 routers, 1 × Cisco 2960 switch, 3 × PCs  
**Protocol:** OSPFv2, process 1, area 0  
**Final assessment:** **100/100** (based on supplied CLI and connectivity evidence)

## Objective

Recreate a Boson-style CCNA lab in which three routers share a broadcast Ethernet segment. Configure OSPF on each router, advertise all directly connected IPv4 networks in area 0, prefer RouterA as designated router (DR), and prevent RouterC from ever becoming DR or backup designated router (BDR).

## Logical topology

```text
            HostA
              |
           RouterA
        Fa0/1 | Fa0/0
              |
           SwitchA
           /     \
       Fa0/0     Fa0/0
       RouterB    RouterC
       Fa0/1      Fa0/1
         |          |
       HostB      HostC
```

RouterA, RouterB, and RouterC connect to the **same Ethernet broadcast domain** using their `FastEthernet0/0` interfaces; SwitchA remains a Layer 2 switch.

## Addressing plan

| Device | Interface | IPv4 address | Subnet |
|---|---|---|---|
| RouterA | Fa0/0 | 10.0.0.1 | 10.0.0.0/24 |
| RouterB | Fa0/0 | 10.0.0.2 | 10.0.0.0/24 |
| RouterC | Fa0/0 | 10.0.0.3 | 10.0.0.0/24 |
| RouterA | Fa0/1 | 192.168.10.1 | 192.168.10.0/24 |
| RouterB | Fa0/1 | 192.168.20.1 | 192.168.20.0/24 |
| RouterC | Fa0/1 | 192.168.30.1 | 192.168.30.0/24 |
| HostA | Fa0 | 192.168.10.10 | 192.168.10.0/24 |
| HostB | Fa0 | 192.168.20.10 | 192.168.20.0/24 |
| HostC | Fa0 | 192.168.30.10 | 192.168.30.0/24 |

Host gateways: `192.168.10.1`, `192.168.20.1`, and `192.168.30.1`, respectively.

## Exam requirements

1. Run OSPF process ID **1** on all three routers.
2. Advertise the specific subnets on each router interface.
3. Put every OSPF-enabled interface in **area 0**.
4. Make RouterA highly preferred as DR on the shared Ethernet segment.
5. Make RouterC ineligible for DR/BDR election.

## Final configuration — relevant commands

These are concise, reproducible configurations. Interface IP addresses were part of the lab's initial setup.

### RouterA

```cisco
hostname RouterA
interface FastEthernet0/0
 ip address 10.0.0.1 255.255.255.0
 ip ospf priority 3
 no shutdown
interface FastEthernet0/1
 ip address 192.168.10.1 255.255.255.0
 no shutdown
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 192.168.10.0 0.0.0.255 area 0
```

RouterA was observed as DR after a new election. Priority **3** was sufficient against the other routers in this exercise; **255** would provide the strongest preference against additional high-priority routers. The user's running configuration also contained `router-id 1.1.1.1`, but operational output initially showed `192.168.10.1`; explicit router ID changes may require an OSPF process restart to take effect.

### RouterB

```cisco
hostname RouterB
interface FastEthernet0/0
 ip address 10.0.0.2 255.255.255.0
 ip ospf priority 2
 no shutdown
interface FastEthernet0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
```

### RouterC

```cisco
hostname RouterC
interface FastEthernet0/0
 ip address 10.0.0.3 255.255.255.0
 ip ospf priority 0
 no shutdown
interface FastEthernet0/1
 ip address 192.168.30.1 255.255.255.0
 no shutdown
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

`ip ospf priority 0` applies only to the relevant interface and means RouterC cannot be elected DR **or BDR** on this shared network.

## Verified OSPF roles

Final neighbor output observed on RouterA:

```text
RouterA#show ip ospf neighbor
Neighbor ID       Pri   State          Address      Interface
3.3.3.3             0   FULL/DROTHER   10.0.0.3     FastEthernet0/0
192.168.20.1        2   FULL/BDR       10.0.0.2     FastEthernet0/0
```

| Router | Priority | Final role | Evidence |
|---|---:|---|---|
| RouterA | 3 | **DR** | Inferred from RouterA's neighbor display (neighbors are BDR and DROTHER) |
| RouterB | 2 | **BDR** | `FULL/BDR` |
| RouterC | 0 | **DROTHER** | `FULL/DROTHER`, priority 0 |

All displayed neighbor adjacencies were `FULL`.

> **Interpretation tip:** `show ip ospf neighbor` lists the **neighbor's role**, not the local router's role. To verify RouterA's own DR state directly, use `show ip ospf interface fastethernet 0/0`.

## Connectivity evidence

Supplied Packet Tracer screenshots demonstrated:

- HostA → `192.168.10.1`: **4/4 replies**.
- RouterA → `10.0.0.2` and `10.0.0.3`: **5/5 replies** each.
- RouterB → `10.0.0.3`: **5/5 replies**.
- HostB → `192.168.20.1`, `192.168.10.1`, `192.168.30.1`: **4/4 replies** for each.
- OSPF tables on every router contained the two remote `/24` LAN routes with OSPF administrative distance **110**.

These tests establish gateway reachability, remote routing, and neighbor relationships. Full host-to-host pings were assigned in the exercise, but not separately documented in the submitted screenshots.

## DR/BDR election lessons

- On broadcast/multiaccess OSPF networks, **highest interface priority** wins an election; **highest router ID** breaks ties.
- Valid priority range is `0–255`; `0` makes the interface ineligible.
- DR/BDR elections are **non-preemptive**: increasing a router's priority does not displace a current DR.
- If the current DR fails, the existing BDR becomes DR, even if a DROTHER has a higher priority.
- Clearing OSPF on one router does not reset the DR election on the other routers. Process restarts may temporarily disrupt traffic.

## Verification commands

```cisco
show ip ospf neighbor
show ip ospf interface fastethernet 0/0
show ip protocols
show ip route
show running-config
```

## Evaluation

| Requirement | Score |
|---|---:|
| OSPF process ID 1 | 15/15 |
| Correct network advertisements | 25/25 |
| OSPF area 0 | 15/15 |
| RouterA DR preference | 20/20 |
| RouterC DR ineligibility | 15/15 |
| Connectivity and verification | 10/10 |
| **Total** | **100/100** |

**Result:** Completed the intended OSPF routing and DR/BDR requirements. The final grade reflects the instructor's assessment based on supplied configurations, routing tables, and Packet Tracer screenshots.
