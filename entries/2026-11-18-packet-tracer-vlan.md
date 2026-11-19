# 2026-11-18 - Packet Tracer: VLANs, inter-VLAN routing and ACLs

**Context:** Designed and built a small company network in Cisco Packet Tracer: an addressing plan for three departments with VLSM, VLANs on a switch, routing between VLANs with a router-on-a-stick, and an extended ACL restricting one department. Lab carried out from 2026-11-16 to 2026-11-18. Devices: Cisco 2911 router, Cisco 2960-24TT switch, six PCs (two per department). Project files and addressing plan: [`projects/cisco-vlan-lab/`](../projects/cisco-vlan-lab/README.md).

**Safety:** Simulated environment. Device configurations saved with `copy running-config startup-config` after each phase, and the whole lab saved as `vlan-lab.pkt`.

**Actions:**

*Addressing plan (on paper)*

- Network `192.168.10.0/24` divided for Uffici (50 devices), Magazzino (20) and IT (10), plus one gateway address each.
- Host bits needed: 6 for Uffici (62 usable), 5 for Magazzino (30), 4 for IT (14), giving `/26`, `/27`, `/28`.
- Blocks assigned from the largest to the smallest (VLSM): `192.168.10.0/26`, `192.168.10.64/27`, `192.168.10.96/28`; `.112`–`.255` left free.
- Masks `255.255.255.192`, `255.255.255.224`, `255.255.255.240`; wildcards `0.0.0.63`, `0.0.0.31`, `0.0.0.15`.

*Switch: VLANs and access ports*

```text
configure terminal
vlan 10
 name UFFICI
vlan 20
 name MAGAZZINO
vlan 30
 name IT
exit
no ip domain-lookup
interface Fa0/2
 switchport access vlan 10
(same for Fa0/3 → 10, Fa0/4-5 → 20, Fa0/6-7 → 30)
interface range Fa0/2 - 7
 switchport mode access
end
show vlan brief
show interfaces fa0/2 switchport
copy running-config startup-config
```

- Before the change, the ports were in VLAN 1 (default) with `Administrative Mode: dynamic auto`; after it, `static access`.
- `show vlan brief`: Fa0/2-3 in VLAN 10, Fa0/4-5 in VLAN 20, Fa0/6-7 in VLAN 30; all other ports still in VLAN 1.

*Switch: trunk to the router*

```text
interface Fa0/1
 switchport mode trunk
end
show interfaces trunk
```

- `show interfaces trunk` was empty until the router interface was enabled: it lists operational trunks only.

*Router: router-on-a-stick*

```text
enable
configure terminal
no ip domain-lookup
interface gig0/0
 no shutdown
interface gig0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.192
interface gig0/0.20
 encapsulation dot1Q 20
 ip address 192.168.10.65 255.255.255.224
interface gig0/0.30
 encapsulation dot1Q 30
 ip address 192.168.10.97 255.255.255.240
end
show ip interface brief
copy running-config startup-config
```

- `Gig0/0` up/up with no IP address; the three subinterfaces up/up with the gateway addresses.
- On the switch, `Fa0/1` now trunking (802.1Q, native VLAN 1) with VLANs 1, 10, 20, 30 active.
- PCs configured with static addresses: Uffici `.2`/`.3`, Magazzino `.66`/`.67`, IT `.98`/`.99`, each with its department mask and gateway.

*Inter-VLAN routing test (before the ACL)*

From Uffici01:

```text
ping 192.168.10.3
ping 192.168.10.1
ping 192.168.10.66
ping 192.168.10.99
```

- All successful. Same VLAN: TTL 128; gateway: TTL 255; other VLANs: TTL 127 (one router crossed).
- The first ping to another VLAN lost one packet while the router resolved the destination MAC (ARP).

*Extended ACL*

```text
configure terminal
ip access-list extended MAGAZZINO-IN
 permit icmp 192.168.10.64 0.0.0.31 192.168.10.0 0.0.0.63 echo-reply
 deny ip 192.168.10.64 0.0.0.31 192.168.10.0 0.0.0.63
 permit ip 192.168.10.64 0.0.0.31 any
exit
interface gig0/0.20
 ip access-group MAGAZZINO-IN in
end
show access-lists
copy running-config startup-config
```

- Policy: Magazzino cannot start connections to Uffici; everything else allowed. Only Magazzino has a restriction, so only its subinterface has an ACL.
- Applied inbound on `Gig0/0.20`, where traffic from Magazzino enters the router.

Tests (after a first invalid round, see Issues):

| Test | Result |
| --- | --- |
| Uffici01 → Magazzino01 (`.66`) | reply from `.66`, TTL 127 |
| Magazzino01 → Uffici01 (`.2`) | `Destination host unreachable` from `.65` |
| Magazzino01 → IT01 (`.98`) | reply from `.98`, TTL 127 |
| IT01 → Magazzino01 (`.66`) | reply from `.66`, TTL 127 |

- `show access-lists` counters: line 10 (`echo-reply`) 4 matches, the replies of the first test; line 20 (deny) 8 matches, the blocked test plus the earlier invalid one towards `192.168.10.0`; line 30 (permit) 8 matches, requests to IT and replies to IT.

*Hardening and regression test*

```text
interface range fa0/8 - 24
 shutdown
interface range gig0/1 - 2
 shutdown
interface Fa0/1
 switchport trunk allowed vlan 10,20,30
end
show interfaces trunk
show interfaces status
```

- Trunk limited to VLANs 10, 20, 30; unused ports `disabled`.
- Regression test: Uffici01 → Magazzino01 still works, Magazzino01 → Uffici01 still blocked.

**Verification:** All four access-policy tests give the expected result, confirmed by the replying address and TTL and by the ACL counters; the regression test after the trunk change passed. Configurations saved on both devices and in `vlan-lab.pkt`.

**Issues:**

- Subnetting on paper: off-by-one errors on block boundaries (broadcast placed one address too far). Fixed with the rule last address = first + 2ⁿ − 1, and blocks starting on a multiple of their size.
- Typed `switchport` and `interface` in privileged mode: an unknown word there is treated as a host name, so the switch tried a DNS lookup and hung for about 30 seconds. Fixed with `configure terminal` first and `no ip domain-lookup`.
- `VLAN 30 name IT` on one line is invalid: `name` belongs to the VLAN sub-mode.
- Answered `y` to `Destination filename [startup-config]?`: the switch tried to save to a file called `y`. The bracketed value is the default; Enter accepts it.
- `show` does not work in configuration mode; `do show` does.
- `switchport trunk allowed 10,20,30` is invalid: the keyword `vlan` is required (found with `?`).
- First ACL test round pinged network addresses (`192.168.10.64`, `.0`, `.96`): the router answered (TTL 255), so the packets never reached the PCs and the ACL was not tested. Repeated against the PCs.

**Lessons learned:**

- Cisco IOS has three main modes: user EXEC (basic commands such as `ping`), privileged EXEC (`enable`: all `show` commands, saving, reloading) and global configuration (`configure terminal`: interfaces, VLANs, ACLs). Changes apply immediately to the running-config and are lost at reboot unless saved to the startup-config.
- A subnet mask splits an address into a fixed part (the prefix, `/N`), which identifies the network, and a free part, which identifies the host inside it. With n free bits a block has 2ⁿ addresses and 2ⁿ − 2 usable ones.
- The first address of a block is the network address and the last one is the broadcast address, which reaches every device in that subnet. Neither can be assigned to a host; the gateway is an ordinary host address (`.1` is only a convention).
- VLSM: subnets of different sizes, assigned from the largest to the smallest so that the address space does not fragment.
- A VLAN separates a switch at layer 2 (one broadcast domain per VLAN). An access port belongs to one VLAN and connects an end device; a trunk carries several VLANs, tagging each frame with its VLAN number (802.1Q), between switches and routers.
- Traffic between VLANs always needs a layer-3 device. In a router-on-a-stick, one physical interface is split into subinterfaces, one per VLAN, each with its tag and the gateway address of its subnet.
- An extended ACL filters on source, destination and protocol. Lines are read top-down and the first match wins; there is an implicit deny at the end, so allowed traffic needs a final `permit`. `in`/`out` are seen from the router: traffic leaving a department enters the router on that department's subinterface.
- Basic Cisco ACLs are stateless, unlike `ufw`: replies must be permitted explicitly (`echo-reply` is the answer to an `echo-request`). An interface without an ACL lets everything through.
- Reading a ping: check who replied and the TTL. A PC starts at 128 and a router at 255; each router crossed subtracts 1, so 127 means one router in between. A test is only valid if it reaches what it is meant to test.
- After every change, repeat the tests that already passed (regression test).

**Next steps:**

- RAID with `mdadm` on the Ubuntu VM.
