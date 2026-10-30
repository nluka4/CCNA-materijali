# CCNA Practical Lab — VLANs, 802.1Q Trunking & LACP EtherChannel

**Platform:** Cisco Packet Tracer  
**Devices:** 2 × Cisco Catalyst 2960 switches, 4 × PCs  
**Lab type:** CCNA practical exam simulation  
**Assessment:** **95/100** (all six original configuration requirements fulfilled; two extended verification checks pending)

## Objective

Configure **SwitchA** to support two VLANs and a standards-based, actively negotiated Layer 2 EtherChannel. **SwitchB** is preconfigured to simulate an exam environment. Verify same-VLAN communication across the EtherChannel and isolation between VLANs.

## Topology

```text
                 Fa0/1 ================= Fa0/1
          +----------+     LACP Po1    +----------+
          | SwitchA  |                 | SwitchB  |
          +----------+                 +----------+
                 Fa0/2 ================= Fa0/2
                /     \                 /      \
          Fa0/3       Fa0/8         Fa0/3      Fa0/8
           PC1         PC2           PC3        PC4
         VLAN 10     VLAN 20       VLAN 10    VLAN 20
```

**Note:** SwitchA's additional access ports Fa0/4 (VLAN 10) and Fa0/9 (VLAN 20) are configured as required, but have no PC attached in this four-PC practice topology. SwitchB also has Fa0/4 and Fa0/9 allocated to the corresponding VLANs.

### Cabling

| Device | Interface | Peer | Interface |
|---|---|---|---|
| SwitchA | Fa0/1 | SwitchB | Fa0/1 |
| SwitchA | Fa0/2 | SwitchB | Fa0/2 |
| SwitchA | Fa0/3 | PC1 | Fa0 |
| SwitchA | Fa0/8 | PC2 | Fa0 |
| SwitchB | Fa0/3 | PC3 | Fa0 |
| SwitchB | Fa0/8 | PC4 | Fa0 |

### Host addressing

These IPv4 addresses were added for practice connectivity tests; they were not part of the original exam prompt.

| Host | VLAN | IP address | Subnet mask |
|---|---:|---|---|
| PC1 | 10 | 192.168.10.11 | 255.255.255.0 |
| PC2 | 20 | 192.168.20.11 | 255.255.255.0 |
| PC3 | 10 | 192.168.10.12 | 255.255.255.0 |
| PC4 | 20 | 192.168.20.12 | 255.255.255.0 |

No default gateways are needed for the intended same-subnet tests. No inter-VLAN routing is configured.

## Original exam requirements

SwitchA must:

1. Explicitly configure Fa0/3, Fa0/4, Fa0/8, and Fa0/9 as access ports.
2. Assign Fa0/3 and Fa0/4 to VLAN 10.
3. Assign Fa0/8 and Fa0/9 to VLAN 20.
4. Use a standards-based EtherChannel negotiation protocol on Fa0/1 and Fa0/2 (**LACP**).
5. Join both interfaces to **channel-group 1** in **active** negotiation mode.
6. Configure the virtual Port-channel interface as an always-on **802.1Q trunk**.

## Initial setup — SwitchB (preconfigured)

```cisco
configure terminal
hostname SwitchB
vlan 10
 name ENGINEERING
exit
vlan 20
 name SALES
exit
interface range fastethernet 0/3 - 4
 switchport mode access
 switchport access vlan 10
exit
interface range fastethernet 0/8 - 9
 switchport mode access
 switchport access vlan 20
exit
interface range fastethernet 0/1 - 2
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
exit
interface port-channel 1
 switchport mode trunk
 no shutdown
end
write memory
```

## Implemented SwitchA configuration

The following lines reproduce the relevant configuration in the submitted `show running-config` output (not a claim that all commands were originally entered in this order).

```cisco
configure terminal
hostname SwitchA

interface FastEthernet0/1
 switchport trunk allowed vlan 10,20
 switchport mode trunk
 channel-group 1 mode active
exit
interface FastEthernet0/2
 switchport trunk allowed vlan 10,20
 switchport mode trunk
 channel-group 1 mode active
exit

interface Port-channel1
 switchport trunk allowed vlan 10,20
 switchport mode trunk
exit

interface FastEthernet0/3
 switchport access vlan 10
 switchport mode access
exit
interface FastEthernet0/4
 switchport access vlan 10
 switchport mode access
exit
interface FastEthernet0/8
 switchport access vlan 20
 switchport mode access
exit
interface FastEthernet0/9
 switchport access vlan 20
 switchport mode access
end
```

VLAN 10 and VLAN 20 are present in SwitchA's VLAN database. A Catalyst 2960 uses 802.1Q trunk encapsulation; on this model an explicit `switchport trunk encapsulation dot1q` command is unnecessary.

## Verification evidence

### VLAN assignments

`show vlan brief` on SwitchA confirmed:

```text
10   VLAN0010   active    Fa0/3, Fa0/4
20   VLAN0020   active    Fa0/8, Fa0/9
```

### Layer 2 EtherChannel

`show etherchannel summary` on SwitchA:

```text
Group  Port-channel  Protocol  Ports
1      Po1(SU)       LACP      Fa0/1(P) Fa0/2(P)
```

- **S** = Layer 2 Port-channel
- **U** = Port-channel in use
- **P** = Physical interface successfully bundled
- **LACP** = Standards-based IEEE link aggregation negotiation

`show interfaces port-channel 1` reported **up/up**, with both Fa0/1 and Fa0/2 as members and 200,000 Kbit/s aggregate interface bandwidth displayed.

### Trunk

`show interfaces trunk` on SwitchA:

```text
Port  Mode  Encapsulation  Status    Native vlan
Po1   on    802.1q         trunking  1

Vlans allowed on trunk:
Po1   10,20

Vlans in spanning tree forwarding state and not pruned:
Po1   10,20
```

The allowed-VLAN list on SwitchA is restricted to VLANs 10 and 20, which is sufficient for the lab.

### Ping tests

| Test | Observed result | Interpretation |
|---|---|---|
| PC1 → PC3 (`192.168.10.12`) | **4/4 replies, 0% loss** | VLAN 10 crosses the trunk successfully |
| PC1 → PC4 (`192.168.20.12`) | **0/4 replies, 100% loss** | Expected: no inter-VLAN routing |
| PC2 → PC4 (`192.168.20.12`) | **Not submitted** | Still needs verification |
| Disconnect one EtherChannel member | **Not submitted** | Still needs redundancy verification |

A failed inter-VLAN ping in this lab is an expected outcome, not a failure of the trunk.

## Evaluation

| Category | Earned | Max |
|---|---:|---:|
| VLAN creation and assignment | 20 | 20 |
| Explicit access-port configuration | 20 | 20 |
| LACP EtherChannel | 25 | 25 |
| 802.1Q trunk configuration | 20 | 20 |
| Connectivity and verification | 10 | 15 |
| **Total** | **95** | **100** |

**Conclusion:** All six requirements in the original configuration task were met. The overall practice score is **95/100** because PC2-to-PC4 reachability and EtherChannel member-failure tolerance have not yet been demonstrated.

## Final verification commands

```cisco
show vlan brief
show interfaces trunk
show etherchannel summary
show interfaces port-channel 1
show running-config
```

To finish the extended practice requirements, test `ping 192.168.20.12` from PC2, then disable one physical EtherChannel link and confirm that Po1 remains up and same-VLAN reachability persists. Restore the disconnected member afterward.

## Key takeaways

- **LACP `active`** initiates negotiation; `active`/`active` is a valid pairing.
- A **Layer 2 Port-channel** can be an **802.1Q trunk** carrying multiple VLANs.
- `Po1(SU)` with members marked `(P)` is strong evidence that aggregation is working.
- Same-VLAN PCs can communicate across a trunk without a router.
- Different VLANs remain isolated until Layer 3 routing is deliberately added.

