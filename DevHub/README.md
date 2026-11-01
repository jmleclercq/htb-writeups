# DevHub — Hack The Box Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/HTB-DevHub-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black">
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white">
</p>

[Hack The Box — DevHub](https://app.hackthebox.com/machines/DevHub)

---

# Machine Information

| Machine | Platform | Difficulty | OS | Status |
|---------|----------|------------|----|--------|
| DevHub | Hack The Box | Medium | Linux | Retired |

---

# Overview

DevHub is a Linux machine whose attack path involves an exposed MCP Inspector, internal services, a Jupyter instance, and a privileged OPSMCP service. The walkthrough follows the evidence from initial service discovery to root access.

> [!NOTE]
> This writeup is for the retired Hack The Box machine. Replace `TARGET` with the IP assigned to your own lab instance. Secrets and executable exploit payloads are intentionally omitted.

## TL;DR

- Discover the development portal and MCP Inspector.
- Identify command execution through the Inspector's server configuration.
- Use the foothold to enumerate internal services and find Jupyter.
- Identify the Jupyter token exposure and obtain execution as `analyst`.
- Inspect the OPSMCP implementation and discover an undocumented privileged operation.
- Use the returned root SSH key to complete the machine.

---

# Table of Contents

- [Machine Information](#machine-information)
- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
- [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
- [Internal Pivot](#internal-pivot)
- [Privilege Escalation](#privilege-escalation)
- [Attack Path Summary](#attack-path-summary)
- [Lessons Learned](#lessons-learned)
- [References](#references)
- [Conclusion](#conclusion)

---

# Reconnaissance

## Nmap Scan

I began with a full TCP port scan and service detection:

```bash
nmap -sC -sV -Pn -p- --min-rate 5000 -oN nmap.txt TARGET
```

### Results

```text
22/tcp open  ssh
80/tcp open  http
```

The web server redirects to `devhub.htb`. I mapped that hostname to the target IP in `/etc/hosts` before continuing.

---

# Web Enumeration

## Development Portal

The portal describes three internal services:

| Service | Purpose | Exposure |
|---------|---------|----------|
| MCP Inspector | Model Context Protocol debugging | Port 6274 |
| Analytics Dashboard | Jupyter analysis environment | Local port 8888 |
| Code Repository | Internal Git service | Unavailable |

The MCP Inspector was reachable on port 6274 and identified itself as MCPJam Inspector v1.4.2.

## MCP Inspector API

The server-list and CLI-configuration endpoints returned no configured MCP servers. Further examination of the connection API showed that its server configuration accepted a process command and arguments.

---

# Foothold

## Command Execution through MCP Inspector

A controlled test confirmed that the command supplied in the MCP server configuration was launched by the service. An MCP protocol error by itself is not proof of execution; the relevant evidence was the resulting process identity:

```text
uid=999(mcp-dev) gid=988(mcp-dev) groups=988(mcp-dev)
```

This provided an initial foothold as `mcp-dev`. The evidence establishes command execution through the Inspector configuration; it is not sufficient to attribute the behavior to a specific CVE.

---

# Internal Pivot

## Internal Services

From the initial foothold, local service enumeration identified two relevant listeners:

```text
127.0.0.1:8888  JupyterLab
127.0.0.1:5000  OPSMCP API
```

Neither service was directly exposed externally. The running Jupyter process also revealed its authentication token in its command-line arguments. The token is omitted here because it is a credential.

## Jupyter Access as `analyst`

Using the recovered token, it was possible to access the local Jupyter API and identify the `quarterly_analysis.ipynb` notebook. Jupyter kernels execute code with the identity of the account running the service; in this case, the kernel ran as `analyst`.

The notebook service therefore provided a privilege boundary crossing from `mcp-dev` to `analyst`. The Jupyter token and the custom WebSocket client used in the original notes are intentionally not reproduced.

---

# Privilege Escalation

## Inspecting OPSMCP

As `analyst`, I was able to read the OPSMCP source code at `/opt/opsmcp/server.py`. The source revealed:

- An API key stored directly in the application code.
- A set of documented tools.
- Additional administrative tools that were callable but not listed through the normal tool-discovery interface.

The API key is redacted because it is a secret. The key lesson is that hiding an operation from discovery does not enforce authorization.

## Root Access

One undocumented administrative operation returned a root SSH private key. Using that key provided a root shell on the machine. The API request, key material, and connection details are omitted; they contain credentials and would make the chain directly reusable.

The resulting identity confirmed root access:

```text
uid=0(root) gid=0(root) groups=0(root)
```

---

# Attack Path Summary

```text
Development portal
  → MCP Inspector exposed on port 6274
  → Command execution as mcp-dev
  → Internal Jupyter service and exposed token
  → Code execution as analyst
  → OPSMCP source disclosure and undocumented administrative operation
  → Root SSH key disclosure
  → Root access
```

---

# Lessons Learned

1. **Do not expose process-launching interfaces.** MCP server configuration should be restricted to trusted, predefined commands and validated arguments.
2. **Protect service credentials.** Tokens should not be exposed in process arguments or readable configuration.
3. **Isolate internal services.** Services bound to localhost can still be reached by compromised local processes; network boundaries alone do not provide process isolation.
4. **Enforce authorization server-side.** Omitting privileged tools from a discovery response does not prevent callers from invoking them.
5. **Avoid hardcoded API keys.** Store secrets outside source code and rotate them if they are exposed.
6. **Limit privilege and secret access.** A service account should not be able to retrieve root authentication material.

---

# References

- [Hack The Box — DevHub](https://app.hackthebox.com/machines/DevHub)

---

# Conclusion

DevHub chains weak controls across several internal services: command execution through the MCP Inspector, a Jupyter token exposed through process metadata, and an undocumented privileged OPSMCP operation that discloses root credentials. The key lesson is to treat each internal service and credential as part of the attack surface, even when it is not directly reachable from the network.
