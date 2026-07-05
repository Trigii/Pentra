---
title: Service Map
draft: false
tags:
  - active-directory
  - kerberos
  - reference
---

When you request or reuse a Kerberos service ticket (TGS), the **SPN service class** determines which protocol/action the ticket is valid for. This mapping is key when doing [[Pass the Ticket (PtT)]] — the ticket you inject must match the service class of the action you want to perform. It also tells you which SPN to target when [[Kerberoasting]].

| Service class (SPN) | Used for |
| --- | --- |
| `cifs` | [[SMB]] file access, `psexec`, `smbexec` (see [[Impacket]]) |
| `http` / `www` | Web apps, [[WinRM]] / PowerShell Remoting, PSRemoting |
| `ldap` | [[AD DCSync\|DCSync]], LDAP binds, directory queries |
| `host` | [[WMI]] and several RPC-based actions (service creation, scheduled tasks) |
| `mssqlsvc` | [[SQL Server Admin\|MSSQL]] logins (common [[Kerberoasting]] target) |
| `rpcss` | WMI / DCOM operations |
| `wsman` | [[WinRM]] (alternative SPN class alongside `http`) |
| `termsrv` | [[RDP]] sessions |

> [!NOTE]
> A single account can expose several SPNs. When you crack or forge a ticket, generate it for the **exact** service class you need. For example, a [[Silver Ticket]] for `cifs/host.domain.local` grants SMB access, while `host/host.domain.local` is required for WMI-based execution.

> [!TIP]
> With Impacket / Rubeus you often need multiple tickets for full control of a host — e.g. `cifs` for file/`psexec` operations **and** `host`/`rpcss` for WMI. Request them together to avoid falling back to NTLM.

## Related notes

- [[Pass the Ticket (PtT)]] — injecting/using these tickets
- [[Kerberoasting]] — requesting TGS for crackable SPNs
- [[Silver Ticket]] — forging service tickets for a specific SPN
- [[Impacket]] — tooling that consumes these SPNs
