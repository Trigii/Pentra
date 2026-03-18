---
title: AD Enumeration - Credentialed - From Windows
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 

> [!Requirements]
> Valid Domain credentials at any permission level:
> - Domain User's cleartext password
> - NTLM password hash
> - SYSTEM access on a domain-joined host

- AD PowerShell module:
```powershell
PS C:\htb> Get-Module (list available modules)
PS C:\htb> Import-Module ActiveDirectory (import AD module if not available)

PS C:\htb> Get-ADDomain (enumerate basic info about the domain)

PS C:\htb> Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName (list kerberoastable accounts: accounts with the ServicePrincipalName populated)

PS C:\htb> Get-ADTrust -Filter * (list domain trusts)

PS C:\htb> Get-ADGroup -Filter * | select name (list AD group info)
PS C:\htb> Get-ADGroup -Identity "GROUP_NAME" (get more info about a group)
PS C:\htb> Get-ADGroupMember -Identity "GROUP_NAME" (list users that belong to a group)
```

- Snaffler:

> [!Requirements]
> Snaffler must be run:
> - From a domain joined host
> - In a domain user context

Snaffler obtains a list of hosts in the specified domain, enumerates the host for shares and readable dirs, and iterates through each dir readable by our user to hunt interesting files.
```bash
Snaffler.exe -s -d DOMAIN_FQDN -o snaffler.log -v data

Parameters:
-s: print the results on the console
-d: domain to research
-o: output results to a log file
-v: verbosity

```
