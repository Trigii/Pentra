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

- Built-in `net` commands (no tooling required, blends in with normal admin activity — useful as [[AD Enumeration (LOTL)|living-off-the-land]]):
```cmd
C:\> net user /domain                 # list all domain users
C:\> net user USERNAME /domain        # detail a single user (groups, logon hours, last set)
C:\> net group /domain                # list all domain groups
C:\> net group "Domain Admins" /domain  # members of a privileged group
C:\> net accounts /domain             # domain password & lockout policy
```

- PowerView (richer object queries — full command set in [[AD Enumeration with PowerView]]):
```powershell
PS C:\> Import-Module .\PowerView.ps1
PS C:\> Get-DomainUser -Properties samaccountname,description | Where-Object {$_.description}  # passwords in descriptions
PS C:\> Get-DomainComputer -Properties dnshostname,operatingsystem
PS C:\> Get-DomainUser -SPN                       # kerberoastable accounts (feed into [[Kerberoasting]])
PS C:\> Get-DomainUser -PreauthNotRequired        # AS-REP roastable accounts (see [[AS-REP Roasting]])
```

- SharpHound (collect graph data to analyse in [[AD Automatic Enumeration (BloodHound)|BloodHound]]):
```powershell
PS C:\> .\SharpHound.exe -c All -d DOMAIN_FQDN --zipfilename loot
# Then import the resulting .zip into the BloodHound GUI to map attack paths.
```

> [!Tip]
> Cleartext creds, an NTLM hash or a Kerberos ticket unlock the whole credentialed workflow. From here, pivot to targeted attacks: [[Kerberoasting]], [[AS-REP Roasting]], [[AD ACL Enumeration and Abuse]] and [[AD DCSync]].

### Related notes
- [[AD Enumeration - Credentialed - From Linux]] — the equivalent workflow from a Linux attack host.
- [[AD Enumeration with PowerView]] — deep-dive on PowerView queries.
- [[AD Automatic Enumeration (BloodHound)]] — graph-based attack-path discovery.
- [[AD Domain Enumeration]] — domain-wide enumeration objectives.
- [[AD Enumeration (LOTL)]] — living-off-the-land enumeration with native tools.
