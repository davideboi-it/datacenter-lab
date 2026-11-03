# 2026-11-02 - Networking lab on srv-lab-01: Part 1

**Context:** Checked the VM's network configuration, followed by connectivity tests to the gateway, hosts outside the subnet and public addresses. 
Also traced the path of the packets from both the VM and the host PC, after installing `traceroute`. 
Lab carried out from 2026-10-30 to 2026-11-02 on an Ubuntu Server 24.04 LTS VM in VirtualBox (recently updated), with NAT networking. 
The host PC runs the Cloudflare One client (WARP). Everything configured by myself.

**Safety:** Manually recreated a backup of the package mirror configuration file (`~/ubuntu.sources.back`) and verified with `diff` that it differs from the current file only in the mirror address. 
A VirtualBox snapshot was taken during this work but no longer exists; cause not determined.

**Actions:**

*Interfaces and routing*

```bash
ip a
ip route
```

- Two interfaces: `lo` (loopback, `127.0.0.1/8`) and `enp0s3` (VirtualBox virtual NIC, MAC `08:00:27:db:94:60`).
- `enp0s3` has `10.0.2.15/24`, assigned dynamically by the VirtualBox DHCP server (24-hour lease).
- IPv6: a link-local address (`fe80::`), generated from the MAC address, and an `fd17::` address.
- Default gateway: `10.0.2.2`. The route to `10.0.2.0/24` was created by the kernel. `10.0.2.3` is the DNS server provided by DHCP.

*Connectivity*

```bash
ping -c 4 10.0.0.2
ping -c 4 10.0.2.2
ping -c 4 google.com
ping -c 4 10.0.2.15
ping -c 4 1.1.1.1
```

- `10.0.0.2` (mistyped, intended `10.0.2.2`): no reply, 100% packet loss.
- `10.0.2.2` (gateway): 0% packet loss, avg 0.34 ms.
- `google.com` (resolved to `216.58.205.238`): 0% packet loss, avg 19.95 ms.
- `10.0.2.15` (the VM's own address): 0% packet loss, avg 0.07 ms.
- `1.1.1.1` (Cloudflare public DNS): 0% packet loss, avg 12.23 ms.

*Package mirror failure and fix*

Diagnosis:

```bash
sudo apt install traceroute
sudo apt-get update
ping -c 4 193.136.212.166
```

- `apt install` failed: `Could not connect to pt.archive.ubuntu.com:80 (193.136.212.166) - connect (111: Connection refused)`.
- DNS was working: the mirror name resolved to `193.136.212.166`.
- Internet access was working: `security.ubuntu.com` answered (`Hit`) during `apt-get update`, and earlier pings to public hosts succeeded.
- `Connection refused`, not a timeout: something at that address actively rejected the connection on port 80.
- Ping to the mirror IP: no reply (100% packet loss). Inconclusive, since many servers block ICMP.
- The mirror was also unreachable from a phone browser, outside the VM, and still down several hours later (`update` and ping repeated with the same result).
- Conclusion: the failure was on the mirror side, not in the VM or the local network.

Fix:

```bash
grep -lr pt.archive.ubuntu.com /etc/apt/
cd /etc/apt/sources.list.d/
cat ubuntu.sources
cp ubuntu.sources ~/
cd ~/
mv ubuntu.sources ubuntu.sources.back   # backup, see Issues
cd /etc/apt/sources.list.d
sudo nano ubuntu.sources
cat ubuntu.sources
sudo apt-get update
sudo apt install traceroute
sudo apt upgrade
sudo reboot
```

- The mirror is set in `/etc/apt/sources.list.d/ubuntu.sources` (deb822 format, two blocks).
- Changed `URIs` in the first block from `http://pt.archive.ubuntu.com/ubuntu/` to `http://archive.ubuntu.com/ubuntu/`, the main Canonical archive. The `security.ubuntu.com` block was left unchanged.
- `apt-get update` completed, `traceroute` was installed, pending updates were applied with `apt upgrade`, then the VM was rebooted.

*Path tracing*

1. From the VM:

```bash
traceroute 1.1.1.1
traceroute -I 1.1.1.1
sudo traceroute -I 1.1.1.1
```

- `traceroute` (default, UDP probes): hop 1 = `10.0.2.2` (gateway), then no reply up to the 30-hop limit.
- `traceroute -I` without `sudo`: refused (`You do not have enough privileges to use this traceroute method`); raw sockets require root, and the fallback ping socket is not allowed for my group.
- `sudo traceroute -I` (ICMP probes): `1.1.1.1` shown directly at hop 1 (~11 ms); the gateway does not appear. The VirtualBox NAT answers ICMP on behalf of the destination.

2. From the host PC (PowerShell):

```powershell
tracert 1.1.1.1
```

- With WARP on: 1 hop, `1.1.1.1` directly. WARP tunnels all traffic to Cloudflare, so the intermediate routers are hidden.
- With WARP off: 7 hops. Hop 1 = `192.168.1.1` (home router), hops 3–4 = `10.x` private addresses inside the provider network, hop 6 = `172.68.100.13` (public, Cloudflare), hop 7 = `1.1.1.1`. Hops 2 and 5 did not reply but forwarded the packets. Latency ~10 ms from hop 3 onwards: the destination is likely nearby.

**Verification:** `apt-get update` completes without errors using `archive.ubuntu.com`; `traceroute` installed and working; full path to `1.1.1.1` visible from the host with WARP off.

**Issues:**
- Mistyped `10.0.0.2` instead of `10.0.2.2`: the address is outside `10.0.2.0/24`, so the packets followed the default route to the gateway and got no reply.
- Backup: copied the file to `~` and edited the copy instead of the original, then renamed the edited copy as the backup. Restored the original mirror address by hand and verified it with `diff` on 2026-11-03.
- Opened `/etc/apt/sources.list.d/ubuntu.sources` with `nano` without `sudo`: changes could not be saved because the file belongs to root. Fixed with `sudo nano`.
- Command typo: `sudo get update`.
- VirtualBox snapshot: created during this work but no longer exists. The shell history shows no shutdown between `apt-get update` and `apt upgrade`, so when it was taken is uncertain.
- After the upgrade the VM hung during shutdown (black screen for over 5 minutes) and was powered off from VirtualBox. No errors noticed at the next boot; root cause not investigated.

**Lessons learned:**

- VirtualBox NAT defaults: the VM gets `10.0.2.15/24` via DHCP, with gateway `10.0.2.2` and DNS `10.0.2.3`. The host cannot reach the VM directly, which is why SSH needs a port forward (host `2222` to VM `22`).
- Troubleshooting order: gateway, then a public IP, then a name. Changing one variable at a time isolates the faulty layer (for example, DNS).
- Read the error: `Connection refused` means something actively rejected the connection; a timeout means nothing answered. A ping with no reply proves nothing, because ICMP is often blocked.
- Backups: make an exact copy with a distinct name (`.bak`), verify it immediately (`cmp` or `diff`, exit code), and only then edit the original. The same applies to snapshots: check that they appear in the list.
- A faulty mirror is fixed in `/etc/apt/sources.list.d/ubuntu.sources`. It is a configuration change, so back up first; in production, a non-urgent install would wait or go through a change request.
- `traceroute` relies on TTL. By default it uses UDP; `-I` uses ICMP and needs root (raw sockets). The VirtualBox NAT and WARP both hide the real path, for different reasons, so trace from a point outside them.
- Use man to find options (`-c`, `-lr`, `-I`), and use the shell history as a log: it is more reliable than memory when documenting work.

**Next steps (Part 2):**
- `ss -tulpn`: listening ports and services.
- Read the `netplan` configuration.
- Take a snapshot and verify it appears in the list, then switch the VM from NAT to bridged and repeat `traceroute` (check whether bridged traffic bypasses WARP on the host).
