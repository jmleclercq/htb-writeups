# Mirai — Hack The Box Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/HTB-Mirai-red?style=for-the-badge&logo=hackthebox&logoColor=white">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white">
</p>

## Machine Information

- **Machine:** Mirai
- **Platform:** Hack The Box
- **IP:** `10.129.88.225`

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
- A full port scan (`-Pn -p-`) against `10.129.88.225` found six open ports: SSH (22), DNS (53), HTTP/Pi-hole (80), UPnP (1999 and 32469), and Plex Media Server (32400).
- Service banners (OpenSSH 6.7p1 on Debian 5+deb8u3) indicate a Debian 8 system, consistent with a small device such as a Raspberry Pi.
- SSH on port 22 was selected as the lead because default credentials are a frequent misconfiguration on Raspberry Pi devices.
- Authenticating over SSH as `pi` / `raspberry` succeeded, giving access to `user.txt` on the `pi` user's Desktop.
- `sudo -l` returned `(ALL) NOPASSWD: ALL` for `pi`, allowing immediate elevation to root.
- The entry under `/root/root.txt` is a decoy ("backup on my USB stick"); the real root flag was recovered by brute-force scanning the raw blocks of the USB stick `/dev/sdb`, where the deleted file still resided.

---

# Reconnaissance

A complete port scan (`-Pn -p-`) of `10.129.88.225` returned six open ports:

| Port | Service | Version / detail |
|------|---------|------------------|
| 22/tcp | SSH | OpenSSH 6.7p1 Debian 5+deb8u3 |
| 53/tcp | DNS | dnsmasq 2.76 |
| 80/tcp | HTTP | lighttpd 1.4.35, `X-Pi-hole` header (Pi-hole interface) |
| 1999/tcp | UPnP | Platinum UPnP 1.0.5.13 (DLNADOC 1.50) |
| 32400/tcp | HTTP | Plex Media Server |
| 32469/tcp | UPnP | Platinum UPnP 1.0.5.13 (DLNADOC 1.50) |

The OS fingerprint — Debian 8 suggested by OpenSSH 6.7p1 and the package set — together with open DNS and SSH ports, is consistent with a small embedded device such as a Raspberry Pi.

**Why SSH was the lead.** Among the exposed services, SSH on port 22 stood out because default credentials are a well-known misconfiguration on Raspberry Pi devices (the classic `pi` / `raspberry` pair). The other services were either known to require authentication (Plex returned `401 Unauthorized`), appeared to be a management interface not yet validated (Pi-hole on port 80), or were network services with no clear unauthenticated path (dnsmasq, UPnP). Given the device profile, testing default SSH credentials offered the most direct and justified initial foothold.

---

# Enumeration & Exploitation

With SSH identified as the lead, the next step was to test the default Raspberry Pi credentials `pi` / `raspberry` on port 22. The reasoning was straightforward: the banner confirmed OpenSSH 6.7p1 on Debian, and the overall footprint matched a Raspberry Pi-style appliance where the `pi` account is created by default and often left with its factory password.

The SSH login succeeded, establishing an interactive shell as the `pi` user. From there, the user flag was located directly on the Desktop:

`/home/pi/Desktop/user.txt`

No web directory brute-forcing, no authentication against Pi-hole or Plex, and no exploit against dnsmasq or UPnP were needed; the default-credential SSH path provided initial access immediately.

---

# Privilege Escalation

Once logged in as `pi`, the privilege check was performed with `sudo -l`. The output showed that `pi` could run every command as any user without a password:

`(ALL) NOPASSWD: ALL`

This is a complete sudo misconfiguration: the `pi` account has unrestricted, passwordless root access. Elevation was therefore immediate (for example, by spawning a root shell with `sudo su` or `sudo bash`), completing the path from initial access to full system compromise.

---

# Flags
- **user.txt**: `REDACTED`
- **root.txt**: `REDACTED`

---

# Lessons Learned

- **Default credentials are a critical foothold.** Embedded and IoT devices ship with well-known default accounts; changing them is the first line of defense.
- **Unrestricted sudo equals root.** Granting `NOPASSWD: ALL` to a standard user removes any privilege boundary; sudo rules should follow least privilege.
- **Decoy flags exist.** The apparent root flag under `/root/root.txt` was a trap pointing to a USB stick; never assume the obvious location holds the real flag.
- **Deleted files can survive.** The root flag was recovered from raw blocks of `/dev/sdb` after deletion, showing that file deletion alone does not erase data.
- **Keep network services patched.** Old versions of dnsmasq (2.76) and Platinum UPnP (1.0.5.13) carry known vulnerabilities (e.g., CVE-2017-14491); even though they were not the entry vector here, they expand the attack surface.

---

# References
- [CVE-2017-14491](https://nvd.nist.gov/vuln/detail/CVE-2017-14491)

---

# Conclusion

Mirai is an Easy Linux box whose entire compromise chain rests on misconfiguration rather than exploitation. A port scan fingerprinted a Debian 8–class device (likely a Raspberry Pi) exposing SSH, DNS, Pi-hole, Plex, and UPnP. The working path was SSH with the default `pi` / `raspberry` credentials, which yielded the user flag and an account with passwordless full sudo rights (`(ALL) NOPASSWD: ALL`) for instant root access. The root flag itself required one extra step: the copy under `/root` was a decoy, and the genuine flag was recovered by scanning the raw blocks of a USB stick where the deleted file persisted. The case illustrates how default credentials and loose sudo policies can fully compromise a device, and how deleted data may remain recoverable from underlying disk blocks.