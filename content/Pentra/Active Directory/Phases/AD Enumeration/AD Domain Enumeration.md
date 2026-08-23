---
title: AD Domain Enumeration
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
- PowerView:
```powershell
PS> Get-NetDomain (basic info about the domain)
PS> Get-Domain (get domain info, including parent domain)
PS> Get-Domain -Domain FQDN_DOMAIN (get info about the specified domain. For example: parent domain)
PS> Get-DomainSID (get curren domain SID)
PS> Get-DomainPolicy (get domain policy about system access or kerberos)
PS> (Get-DomainPolicy)."SystemAccess" (get domain policy about system access)
PS> (Get-DomainPolicy)."kerberospolicy" (get domain policy about kerberos)
```

- Basic domain info enumeration:
```powershell
PS C:\htb> Get-Module # list available modules
PS C:\htb> Import-Module ActiveDirectory # import AD module if not available

PS C:\htb> Get-ADDomain # enumerate basic info about the domain
```

- Enumerate Domain Object for the current user:
```powershell
PS C:\> [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

> [!Note]
> This contains the PdcRoleOwner property, which indicates us who is the Primary DC

- Enumerate Distinguished Name (DN) for the current Domain:
```powershell
PS C:\> ([adsi]'').distinguishedName
```

# AD Domain Enumeration Script

- Full LDAP enumeration:
```bash
$ ldapsearch -v -x -b “DC=DOMAIN_CN,DC=DOMAIN_NAME” -H “ldap://TARGET_IP” “(objectclass=*)”
```

- Users + Description Enumeration (descriptions frequently leak passwords or hints):
```bash
# ldapsearch: dump every user's sAMAccountName and description in one query
$ ldapsearch -x -H ldap://DC_IP -D "DOMAIN\\USER" -w 'PASSWORD' \
    -b "DC=DOMAIN_CN,DC=DOMAIN_NAME" \
    "(&(objectClass=user)(objectCategory=person))" sAMAccountName description

# NetExec / CrackMapExec: quick user + description sweep
$ netexec ldap DC_IP -u USER -p 'PASSWORD' --users

# PowerView (from a domain-joined Windows host)
PS C:\> Get-DomainUser * | Select-Object samaccountname, description | Where-Object {$_.description}
```

> [!Tip]
> Descriptions are a classic quick win — grep the output for `pass`, `pwd`, `temp`, `welcome`. Feed any hits straight into [[AD Password Spraying]].

- Script:
```powershell
function LDAPSearch {
    param (
        [string]$LDAPQuery
    )

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DistinguishedName = ([adsi]'').distinguishedName

    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DistinguishedName")

    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)

    return $DirectorySearcher.FindAll()

}
```

> [!Important]
> This script enumerates further than tools like `net` because is queries the Primary DC.

- Usage:
```powershell
PS C:\> Import-Module .\<SCRIPT>.ps1 # import function

PS C:\> LDAPSearch -LDAPQuery "(samAccountType=805306368)" # get all users in the domain

PS C:\> LDAPSearch -LDAPQuery "(objectclass=group)" # enumerate all groups in the domain

# enumerate one specific group
PS C:\> $sales = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=<GROUP_NAME>))"
PS C:\> $sales.properties.member


PS C:\> foreach ($group in $(LDAPSearch -LDAPQuery "(objectCategory=group)")) {
$group.properties | select {$_.cn}, {$_.member}
} # display all groups in the domain and users for each group
```

# Enumerate DNS Records
```bash
$ adidnsdump -u DOMAIN_NAME\\USERNAME ldap://DC_IP -r # enumerate records

$ head records.csv # view the contents
```

---

### Related notes
- [[AD Enumeration with PowerView]] — deeper PowerView-based enumeration.
- [[AD Enumerating Users]] — user-focused enumeration and validation.
- [[AD Automatic Enumeration (BloodHound)]] — graph the whole domain automatically.
- [[AD Enumeration - Credentialed - From Linux]] — the Linux-side equivalent (ldapsearch, NetExec, rpcclient).
- [[AD Password Spraying]] — natural next step once you have a user list.
