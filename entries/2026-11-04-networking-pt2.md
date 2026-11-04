# 2026-11-04 - Networking lab on srv-lab-01: Part 2

**Context:** Second part of the networking lab. Checked listening ports and services, followed the DNS resolution chain inside the VM, read the netplan configuration, then switched the VM from NAT to bridged networking to test the predictions from Part 1 (real path visible from the VM, traffic bypassing WARP on the host). Rolled back to NAT at the end. Lab carried out on 2026-11-04 on the same Ubuntu Server 24.04 LTS VM in VirtualBox, with the Cloudflare One client (WARP) active on the host PC the whole day.

**Safety:** VirtualBox snapshot "PreBridged04/11/26" taken with the VM powered off and verified in the snapshot list before changing the network mode. Used to roll back to NAT, then deleted after the rollback was verified.

**Actions:**

*Listening sockets*

```bash
ss -tulpn
sudo ss -tulpn
```

- TCP `22` listening on `0.0.0.0` and `[::]` (all IPv4 and IPv6 interfaces): `sshd`, together with `systemd` (PID 1, socket activation).
- TCP and UDP `53` listening on `127.0.0.53%lo` and `127.0.0.54`: `systemd-resolved`.
- UDP `68` on `10.0.2.15%enp0s3`: `systemd-networkd` (DHCP client).
- Without `sudo` the `Process` column is empty; with `sudo` the process names and PIDs are shown.

*DNS resolution chain*

```bash
resolvectl status
ls -l resolv.conf
cat resolv.conf
```

- `resolvectl status`: `enp0s3` uses DNS server `10.0.2.3` (plus an IPv6 one, `fd17:625c:f037:2::3`), DNS domain `home`; `resolv.conf mode: stub`.
- `/etc/resolv.conf` is a symbolic link to `../run/systemd/resolve/stub-resolv.conf`, which contains `nameserver 127.0.0.53` and `search home`.
- Chain: programs → `/etc/resolv.conf` → `127.0.0.53` (`systemd-resolved`) → `10.0.2.3` via `enp0s3` (VirtualBox) → DNS of the host PC.

*Netplan configuration*

```bash
cd /etc/netplan/
ls -la
sudo cat 50-cloud-init.yaml
```

- One file, `50-cloud-init.yaml`, permissions `-rw-------` (root only), so `sudo` is required to read it.
- Content: `enp0s3` with `dhcp4: true` only. Address, gateway, DNS and search domain all come from DHCP.

*Bridged mode test*

VirtualBox settings → Network → Adapter 1: attached to "Bridged Adapter" on the physical NIC (Intel Ethernet Connection I219-V). Logged in from the VirtualBox console, then over SSH.

```bash
ip a
```

```powershell
ssh boi@192.168.1.6 -p 22
```

```bas
traceroute 1.1.1.1
sudo traceroute -I 1.1.1.1
```

- `enp0s3` got `192.168.1.6/24` from the home router's DHCP (lease about 48 hours) and a public IPv6 address (`2a01:...`) via SLAAC. The MAC address did not change.
- The netplan file did not need any change.
- SSH: the `2222` port forward no longer applies; connected directly to `192.168.1.6` on port 22. SSH recognised the same host key previously stored as `[127.0.0.1]:2222`.
- Both `traceroute` methods (UDP and ICMP) showed the full path in 7 hops, with WARP active on the host: `192.168.1.1` (home router), provider `10.x` addresses, `172.68.100.13` (Cloudflare), `1.1.1.1`. Same path as `tracert` from the host with WARP off.
- Hop 4 (UDP run) answered from two different routers, `10.255.48.74` and `10.255.48.78`.

*Rollback to NAT*

VM powered off, snapshot "PreBridged04/11/26" restored from VirtualBox.

```bash
ip a
```

- `enp0s3` back to `10.0.2.15/24` with the `fd17::` IPv6 address; SSH again through `127.0.0.1:2222`.

**Verification:** After the rollback, `ip a` shows the original NAT configuration and SSH through the port forward works as before.

**Issues:** None during this part.

**Lessons learned:**

- `ss -tulpn` lists listening TCP and UDP sockets with numeric ports. Process names need `sudo`, because those processes belong to root (similar to `traceroute -I`).
- In NAT the VM is the DHCP client (UDP 68); the DHCP server (UDP 67) is VirtualBox, outside the VM.
- The listening address decides who can connect: `0.0.0.0` / `[::]` means all interfaces (SSH is reachable from outside the VM), while `127.0.0.53%lo` is loopback only (local programs only).
- DNS in the VM: programs ask `127.0.0.53`, a local stub run by `systemd-resolved` with its own cache, which forwards to the DNS server received via DHCP (`10.0.2.3`) through `enp0s3`. Listening on loopback does not stop it from asking servers outside.
- TCP sockets show `LISTEN` because TCP is connection-oriented and waits for connections; UDP is connectionless, so its sockets are simply open (`UNCONN`).
- Do not edit `/etc/resolv.conf` by hand: it is a symlink to a file in `/run` that `systemd-resolved` regenerates, so edits are lost, and replacing the link breaks the automatic management. Change the configuration at its source.
- Netplan stores the network configuration in `/etc/netplan/` (YAML, root-only because it can contain passwords). With DHCP it only needs `dhcp4: true`, and the same file works in both NAT and bridged mode.
- In bridged mode the VM is a normal device on the home network: it gets its IPv4 address from the router's DHCP and a public IPv6 address via SLAAC. Whether it is reachable from the internet depends on the router's firewall, which usually blocks inbound connections by default; this must be checked, not assumed.
- The hops hidden in Part 1 were caused by the VirtualBox NAT, not by the protocol: in bridged mode both UDP and ICMP `traceroute` show the full path, and the VM's traffic bypasses WARP on the host.
- Two routers answering at the same hop means load balancing across parallel links (active-active), which also gives redundancy.
- A VM with password-based SSH should not be exposed on a shared network. Rolled back to NAT via the snapshot until SSH is hardened.

**Next steps:**

- SSH hardening project: `/etc/ssh/sshd_config`, key-only authentication, firewall rules.
- After that, consider bridged mode permanently, with a DHCP reservation on the router or a static address in netplan.
