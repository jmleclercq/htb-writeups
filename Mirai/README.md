# Mirai — Hack The Box Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/HTB-Mirai-red?style=for-the-badge&logo=hackthebox&logoColor=white">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white">
</p>

## Machine Information

- **Machine:** Mirai
- **Platform:** Hack The Box
- **IP:** `<IP_Machine>`

---

# Table of Contents

- [TL;DR](#tldr)
- [Reconnaissance](#reconnaissance)
- [Enumeration & Exploitation](#enumeration--exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [References](#references)
- [Conclusion](#conclusion)

---

# TL;DR

- Full port scan reveals a Raspberry Pi running Pi-hole (port 80), SSH (port 22), DNS via dnsmasq, and a Plex Media Server.
- The machine's identity as a Raspberry Pi strongly suggests the use of default credentials.
- SSH login succeeds immediately with the default Raspberry Pi OS credentials (`pi` / `raspberry`).
- The `pi` user has full passwordless sudo rights, granting instant root access.
- `user.txt` is found directly on the pi user's Desktop.
- `root.txt` has been deliberately deleted from a USB stick — but is recovered by reading raw blocks from the underlying block device (`/dev/sdb`).

---

# Reconnaissance

The initial reconnaissance begins with a full TCP port scan to get a complete picture of the attack surface:

```bash
nmap -Pn -p- <IP_Machine>
```

Followed by a service and version detection pass on the discovered open ports:

```bash
nmap -Pn -sV -p 22,53,80,1999,32400,32469 <IP_Machine>
```

The results reveal six open ports:

| Port | Service | Detail |
|---|---|---|
| 22/tcp | SSH | OpenSSH 6.7p1 Debian 5+deb8u3 |
| 53/tcp | DNS | dnsmasq 2.76 |
| 80/tcp | HTTP | lighttpd 1.4.35, `X-Pi-hole` header present |
| 1999/tcp | UPnP | Platinum UPnP 1.0.5.13 (DLNADOC 1.50) |
| 32400/tcp | Plex | Plex Media Server (responds 401 Unauthorized) |
| 32469/tcp | UPnP | Platinum UPnP 1.0.5.13 (DLNADOC 1.50) |

The combination of services here tells a very clear story. The presence of **dnsmasq** as a DNS resolver, **Pi-hole** on port 80 (confirmed by the `X-Pi-hole` HTTP response header and the lighttpd web server), and **Plex Media Server** on port 32400 is a near-perfect fingerprint of a **Raspberry Pi** running Raspberry Pi OS. This is not a coincidence — the machine is named "Mirai", a reference to the infamous botnet that compromised hundreds of thousands of IoT devices, including Raspberry Pis, by exploiting default credentials.

The SSH version (OpenSSH 6.7p1, Debian 5+deb8u3) is consistent with Debian 8 (Jessie), the distribution typically shipped on older Raspberry Pi images.

The most promising lead is clear: **try the default Raspberry Pi OS credentials over SSH before anything else**.

---

# Enumeration & Exploitation

With the hypothesis that this is a Raspberry Pi using its factory default credentials, the exploitation step is straightforward:

```bash
ssh pi@<IP_Machine>
```

When prompted for a password, the default Raspberry Pi OS password `raspberry` is entered. The login succeeds immediately:

```
pi@raspberrypi:~ $
```

No brute-forcing, no vulnerability exploitation, no web attack — the machine was simply left with the credentials it shipped with. This is precisely the class of vulnerability Mirai-type botnets exploit at scale.

Once logged in, the user flag is immediately accessible on the pi user's Desktop:

```bash
cat /home/pi/Desktop/user.txt
```

---

# Privilege Escalation

Checking the sudo permissions for the `pi` user:

```bash
sudo -l
```

The output reveals the following:

```
(ALL) NOPASSWD: ALL
```

The `pi` user has unrestricted, passwordless sudo access over the entire system. This is another Raspberry Pi OS default: the `pi` account is configured to run any command as root without requiring a password. Escalation is a single command:

```bash
sudo su -
root@raspberrypi:~#
```

Root access is now obtained. However, reading `/root/root.txt` reveals an unexpected obstacle:

```
I lost my original root.txt! I think I may have a backup on my USB stick...
```

This is a deliberate challenge twist. The real flag was stored on a USB stick, which is mounted at `/media/usbstick` and backed by the block device `/dev/sdb`. The USB stick's files have been deleted, so a standard `ls` or `cat` yields nothing useful.

The file is gone from the filesystem, but its contents may still reside in the unallocated blocks of the device. Since the device is small and we have root access, we can read the raw block data directly:

```bash
strings /dev/sdb
```

The `strings` command scans the raw bytes of the block device and extracts any printable ASCII sequences. Among the output, the root flag value appears — it was never truly erased from the underlying storage, only unlinked from the filesystem.

---

# Flags

- **user.txt**: `REDACTED`
- **root.txt**: `REDACTED`

---

# Lessons Learned

1. **Default credentials are never acceptable on internet-facing or network-accessible devices.** The `pi` / `raspberry` combination is publicly documented. Any device shipped with known default credentials and exposed to a network is trivially compromised, exactly as Mirai demonstrated at internet scale in 2016.

2. **The principle of least privilege must apply even to "trusted" local accounts.** Granting a standard user account full passwordless `sudo` rights eliminates any meaningful access control boundary. If SSH is compromised, root is immediately compromised too.

3. **Deleting a file does not erase its data.** Unlinking a file from a filesystem only removes the directory entry and marks the blocks as available for reuse. Until those blocks are overwritten, the data remains recoverable with tools like `strings`, `foremost`, or `photorec`. Sensitive data must be securely wiped (e.g., with `shred` or `dd` with `/dev/zero`) before a device is decommissioned or shared.

4. **Service fingerprinting is a powerful enumeration technique.** The combination of Pi-hole, dnsmasq, lighttpd, and Plex is a near-unambiguous indicator of a Raspberry Pi. Knowing the target platform immediately narrows down the most likely misconfigurations to test.

5. **IoT and embedded devices require the same hardening discipline as servers.** Changing default passwords, restricting sudo, and disabling unnecessary services are baseline requirements regardless of the device's perceived importance.

---

# References

- [CVE-2017-14491](https://nvd.nist.gov/vuln/detail/CVE-2017-14491) — dnsmasq heap overflow (dnsmasq < 2.78, DNS responses) — version 2.76 present on this machine is within the affected range.

---

# Conclusion

Mirai is a machine that teaches through simplicity. The entire attack chain — from reconnaissance to root flag — requires no exploit, no CVE, and no advanced technique. The machine is compromised because it runs with the exact same default configuration that millions of Raspberry Pi devices ship with: a well-known default password for a user account that has unconditional root access. The final twist, recovering a deleted flag from raw disk blocks, adds a practical forensics lesson on top of the security fundamentals. The machine's name is a direct tribute to the Mirai botnet, a reminder that some of the most devastating attacks in history were built not on zero-days, but on unchanged factory defaults.