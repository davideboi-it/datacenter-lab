# Packet Tracer: VLANs, inter-VLAN routing and ACLs

A small company network built in Cisco Packet Tracer: an addressing plan with VLSM, three departments separated into VLANs on one switch, routing between them with a router-on-a-stick, and an extended ACL that restricts one department.

File: `vlan-lab.pkt` (Cisco Packet Tracer).

## Topology

```text
                 Router 2911
                 Gig0/0 (subinterfaces .10 .20 .30)
                    |
                    | trunk 802.1Q (VLANs 10, 20, 30)
                    |
                 Fa0/1
              Switch 2960-24TT
   Fa0/2  Fa0/3   Fa0/4  Fa0/5     Fa0/6  Fa0/7
     |      |       |      |         |      |
 Uffici01 Uffici02  Magazzino01/02    IT01   IT02
   (VLAN 10)          (VLAN 20)       (VLAN 30)
```

Unused switch ports (`Fa0/8-24`, `Gig0/1-2`) are shut down.

## Addressing plan

Network: `192.168.10.0/24`, divided with VLSM from the largest to the smallest subnet.

| Department | Devices | VLAN | Subnet | Mask | Usable | Gateway | Broadcast |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Uffici | 50 | 10 | `192.168.10.0/26` | `255.255.255.192` | `.1` – `.62` | `.1` | `.63` |
| Magazzino | 20 | 20 | `192.168.10.64/27` | `255.255.255.224` | `.65` – `.94` | `.65` | `.95` |
| IT | 10 | 30 | `192.168.10.96/28` | `255.255.255.240` | `.97` – `.110` | `.97` | `.111` |
| Free | | | `192.168.10.112` – `.255` | | | | |

| Host | IP | Gateway |
| --- | --- | --- |
| Uffici01 | `192.168.10.2` | `192.168.10.1` |
| Uffici02 | `192.168.10.3` | `192.168.10.1` |
| Magazzino01 | `192.168.10.66` | `192.168.10.65` |
| Magazzino02 | `192.168.10.67` | `192.168.10.65` |
| IT01 | `192.168.10.98` | `192.168.10.97` |
| IT02 | `192.168.10.99` | `192.168.10.97` |

## Configuration summary

**Switch**
- VLANs 10 `UFFICI`, 20 `MAGAZZINO`, 30 `IT`.
- PC ports in static access mode, each in its department VLAN.
- `Fa0/1` trunk to the router, allowed VLANs limited to `10,20,30`.
- Unused ports shut down; `no ip domain-lookup`.

**Router**
- `Gig0/0` enabled, no IP address.
- Subinterfaces `Gig0/0.10`, `.20`, `.30` with `encapsulation dot1Q` and the department gateway address.
- Extended ACL `MAGAZZINO-IN` applied inbound on `Gig0/0.20`.

## Access policy

| From ↓ / To → | Uffici | Magazzino | IT |
| --- | --- | --- | --- |
| Uffici | — | allowed | allowed |
| Magazzino | **blocked** (replies only) | — | allowed |
| IT | allowed | allowed | — |

```text
ip access-list extended MAGAZZINO-IN
 permit icmp 192.168.10.64 0.0.0.31 192.168.10.0 0.0.0.63 echo-reply
 deny ip 192.168.10.64 0.0.0.31 192.168.10.0 0.0.0.63
 permit ip 192.168.10.64 0.0.0.31 any
!
interface GigabitEthernet0/0.20
 ip access-group MAGAZZINO-IN in
```

The ACL is stateless, so the replies from Magazzino to Uffici (`echo-reply`) are permitted explicitly before the deny line.

## Verification

| Test | Expected | Result |
| --- | --- | --- |
| Uffici01 → Magazzino01 (`.66`) | allowed | reply from `.66`, TTL 127 |
| Magazzino01 → Uffici01 (`.2`) | blocked | `Destination host unreachable` from `.65` |
| Magazzino01 → IT01 (`.98`) | allowed | reply from `.98`, TTL 127 |
| IT01 → Magazzino01 (`.66`) | allowed | reply from `.66`, TTL 127 |

The same tests were repeated after limiting the trunk VLANs (regression test).

Lab diary entry: [`entries/2026-11-18-packet-tracer-vlan.md`](../../entries/2026-11-18-packet-tracer-vlan.md).
