# 2026-11-14 - SSH hardening on srv-lab-01

**Context:** Hardened SSH access to the VM: key-only authentication, no direct root login, and a host firewall that allows only SSH. Then tested from the home network in bridged mode, and went back to NAT. Lab carried out from 2026-11-05 to 2026-11-14 (with a one-week pause) on the Ubuntu Server 24.04 LTS VM in VirtualBox, NAT networking, client: Windows PC with OpenSSH.

**Safety:** VirtualBox snapshot "PreNewSSHKey" taken with the VM powered off and verified in the snapshot list before any change. An existing SSH session was kept open during every `sshd` restart and while enabling the firewall; the VirtualBox console was the fallback access. Old configuration backups are kept in `~/backups/` on the VM, outside the repository.

**Actions:**

*Initial state*

```bash
ls -la ~/.ssh
ls -la /etc/ssh
ssh-keygen -lf ssh_host_ed25519_key.pub
grep -i "password" sshd_config
ls -la sshd_config.d
sudo cat 50-cloud-init.conf
grep -inr sshd_config.d
sudo sshd -T | grep -i "passwordauthentication"
```

- `~/.ssh/authorized_keys` existed but was empty (0 bytes): login was password-only. The `id_ed25519` pair in `~/.ssh` is the VM's own client key for GitHub.
- `/etc/ssh/` holds three host key pairs (`ed25519`, `ecdsa`, `rsa`). The `ed25519` fingerprint (`SHA256:ziGx6ws...`) matched the one shown by the SSH client at the first bridged connection.
- In `sshd_config`, `PasswordAuthentication yes` is commented out (it documents the default). `/etc/ssh/sshd_config.d/50-cloud-init.conf`, written by cloud-init, contains `PasswordAuthentication yes`.
- `sshd_config` line 12: `Include /etc/ssh/sshd_config.d/*.conf`. For `sshd` the first value read wins, so the drop-in files override anything set later in `sshd_config`.
- Effective value: `passwordauthentication yes`.

*Key-based login*

On the PC (PowerShell):

```powershell
ssh-keygen -t ed25519 -C "boi@pcwin"
scp -P 2222 "C:\Users\ElBiguz Boi\.ssh\id_ed25519.pub" boi@127.0.0.1:
```

On the VM:

```bash
cat id_ed25519.pub >> ~/.ssh/authorized_keys
ls -la
cat authorized_keys
```

Test from a second PowerShell window:

```powershell
ssh -v boi@127.0.0.1 -p 2222
```

- New ED25519 key pair on the PC, protected by a passphrase; fingerprint `SHA256:2mtBi/cE2...`.
- Public key appended with `>>`; `authorized_keys` is 92 bytes (same as the `.pub` file) with permissions `-rw-------`.
- `ssh -v`: the client verified the server's host key against `known_hosts`, the server listed `publickey,password`, accepted the key, and the passphrase was asked locally. Result: `Authenticated ... using "publickey"`.

*Disabling password and root login*

```bash
sudo nano 01-hardening.conf
sudo sshd -t
sudo sshd -T | grep -i "password"
sudo systemctl restart ssh
sudo systemctl status ssh
```

- Created `/etc/ssh/sshd_config.d/01-hardening.conf`, read before `50-cloud-init.conf`, with `PasswordAuthentication no` and then `PermitRootLogin no`.
- `sshd -t`: no output (syntax OK). `sshd -T`: `passwordauthentication no`, `permitrootlogin no`, read from the files on disk before the restart.
- `ssh.service` restarted; `systemctl status` shows `ExecStartPre=/usr/sbin/sshd -t (status=0/SUCCESS)` and `TriggeredBy: ssh.socket`. The open session stayed up.

Tests from a new PowerShell window:

```powershell
ssh boi@127.0.0.1 -p 2222
ssh -o PubkeyAuthentication=no boi@127.0.0.1 -p 2222
```

- With the key: only the passphrase is asked.
- Without the key: `Permission denied (publickey)`. The server now offers only `publickey`.

*Logs*

```bash
journalctl -n 25
```

- `Accepted publickey for boi from 10.0.2.2 ... ED25519 SHA256:2mtBi/cE2...`: the log records which key was used.
- The test without key: `Connection reset by authenticating user boi 10.0.2.2 ... [preauth]`, closed by the client before authentication, since no password method was offered.
- Every connection comes from `10.0.2.2`: the VirtualBox NAT hides the real client address.

*Firewall*

```bash
sudo ufw status
sudo grep -A 5 "dhcp" before.rules
sudo grep -A 5 "loopback" before.rules
sudo grep -A 5 "echo-request" before.rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw show added
sudo ufw enable
sudo ufw status verbose
```

- `ufw` was installed but inactive.
- `/etc/ufw/before.rules` always accepts DHCP client replies (UDP 67 to 68), loopback traffic, ping (`echo-request`) and packets of established connections (`RELATED,ESTABLISHED`).
- Policies: deny incoming, allow outgoing. Only `22/tcp` allowed (IPv4 and IPv6), added before enabling the firewall.
- Firewall active and enabled at boot; the open session stayed up and a new key login worked.

*Test in bridged mode*

VirtualBox network adapter switched to bridged (Intel I219-V); the VM got `192.168.1.6/24` again from the home router.

```powershell
ssh boi@192.168.1.6 -p 22
ssh -o PubkeyAuthentication=no boi@192.168.1.6 -p 22
```

- Key login works; without key: `Permission denied (publickey)`.

On the VM, a temporary web server on a port not allowed by the firewall, started from an empty folder:

```bash
mkdir test1
cd test1/
python3 -m http.server 8080
```

In a second session:

```bash
ss -tulpn
```

From the PC:

```powershell
Test-NetConnection 192.168.1.6 -Port 8080
```

- Port `8080` confirmed listening on `0.0.0.0` (`python3`).
- Firewall on: `TcpTestSucceeded: False` after about 10 seconds; `PingSucceeded: True` (ping is allowed by `before.rules`).
- Counter-test, one variable changed: `sudo ufw disable` → `TcpTestSucceeded: True`; then `sudo ufw enable` and `sudo ufw status verbose` again.

```bash
journalctl -k | grep "UFW BLOCK"
```

- Five `[UFW BLOCK]` lines for the test: `SRC=192.168.1.12` (the PC) `DST=192.168.1.6`, `DPT=8080`, `SYN`, `TTL=128` (Windows). The retries came after 1, 2, 4 and 8 seconds, which explains the ~10-second wait: the firewall drops packets silently instead of refusing them.
- Other blocked packets every ~2 minutes: IGMP (`PROTO=2`) multicast queries to `224.0.0.1`, most likely from the home router.

*Back to NAT and cleanup*

- Python server stopped, `test1` removed, adapter set back to NAT, key login through `127.0.0.1:2222` working with the firewall on.
- Removed the copied `~/id_ed25519.pub` from the VM home.

**Verification:** `sshd -T` shows `passwordauthentication no` and `permitrootlogin no`; `ufw status verbose` shows deny incoming / allow outgoing with only `22/tcp` allowed; key login works in NAT and bridged mode, password login is refused, and a non-allowed port is blocked (confirmed by the counter-test with the firewall off).

**Issues:**

- Used `||` instead of `|` with `sshd -T`: `grep` never ran because the first command succeeded. Fixed with the pipe.
- Ran `sshd -t` and `sshd -T` without `sudo`: `Permission denied` on `50-cloud-init.conf`, which only root can read.
- First `scp` attempts: missing `:` after the host, `-P` written after the file names and the wrong port. Another attempt failed with `Connection refused` because the VM was powered off (nothing listening on port 2222).
- The lab was left half-finished for a week, so part of the context had to be rebuilt.

**Lessons learned:**

- A key pair is two complete, linked keys. The private key stays on the client and never travels; the server only holds the public key, in `authorized_keys`. At login the client proves it owns the private key, and the server checks it with the public one.
- Two trust files, one per direction: `known_hosts` on the client stores the servers' host keys (the client trusts the server); `authorized_keys` on the server stores the public keys allowed to log in (the server trusts the client).
- `sshd_config` includes `sshd_config.d/*.conf` at line 12, in alphabetical order, and the first value read wins. An option already set in a drop-in file cannot be overridden later in `sshd_config`, so my rules go in my own early file (`01-hardening.conf`).
- `sshd -t` checks the syntax (no output means OK); `sshd -T` prints the full effective configuration computed from the files on disk, i.e. what will apply after a restart. The running service keeps its old configuration until it is restarted.
- Safe order: create the key on the client, install the public key on the server, test the key login, then disable passwords. Keep a session open while restarting `sshd`, and test from a new session.
- `PermitRootLogin no` blocks SSH logins as root entirely, even with a key. `root` exists on every Linux system, so it is the first target; admins log in with their own user and use `sudo`, which also records who did what.
- A stateful firewall only needs incoming rules for services the server offers (here, SSH). DNS, HTTPS and package updates are outgoing connections started by the VM, and their replies are allowed by the `ESTABLISHED` rule.
- A test is only useful if it can fail. Logging in as root, or reaching port 8080 in NAT, would fail even without the change; the real proof is `sshd -T`, and for the firewall a counter-test changing one variable (`ufw disable`).
- `ufw` drops blocked packets silently: the client waits and retries until it times out, instead of getting an immediate `Connection refused`.
- `[preauth]` in the SSH log means the connection ended before authentication, not that the client was pre-authorised. `UFW BLOCK` lines show source, destination, ports and protocol of each blocked packet.
- `fail2ban` bans IP addresses after repeated failed logins. Behind the VirtualBox NAT every connection comes from `10.0.2.2`, so a ban would lock out everyone, including me; trusted addresses go in its `ignoreip` list.

**Next steps:**

- Delete the "PreNewSSHKey" snapshot after a few stable days and check it is gone from the list.
- Optional: enable `ssh-agent` on Windows so the passphrase is not asked at every login.
- Any new device (laptop, phone) gets its own key added to `authorized_keys`. A permanent bridged setup would need a static address in netplan, outside the router's DHCP range (the router is managed by the company).
- Next project: Packet Tracer (subnetting, VLAN, routing, ACL).
