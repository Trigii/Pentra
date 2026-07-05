---
title: AD Enumerating Groups
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
# LOTL Tools

- Net:
```powershell
C:\> net group /domain # enumerate ALL domain groups

C:\> net group "GROUP" /domain # enumerate members of a group
```

- Crackmapexec:
```bash
$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --groups # retrieve a list of all domain groups
```

- [Windapsearch](https://github.com/ropnop/windapsearch):
```bash
$ python3 windapsearch.py --dc-ip DC_IP -u USERNAME@DOMAIN_FQDN -p PASSWORD --da # enumerate domain admins group members
```

---
# PowerView

> [!Requirements]
> Valid domain credentials and access to a domain joined host (see [[AD Enumeration with PowerView]]).

```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1

PS> Get-DomainGroup | select samaccountname # list all domain groups
PS> Get-DomainGroup -Identity "Domain Admins" # detailed info about a group
PS> Get-DomainGroupMember -Identity "Domain Admins" -Recurse # members of a group (recursive resolves nested groups)
PS> Get-DomainGroup -MemberIdentity USER # groups a specific user belongs to
```

---
# LDAP (from Linux)

- Using ldapsearch (anonymous bind or with credentials):
```bash
$ ldapsearch -x -H ldap://DC_IP -b "DC=DOMAIN,DC=LOCAL" "(objectClass=group)" sAMAccountName # list groups
$ ldapsearch -x -H ldap://DC_IP -D "USER@DOMAIN_FQDN" -w PASSWORD -b "DC=DOMAIN,DC=LOCAL" "(cn=Domain Admins)" member # members of a group
```

---
# Automatic Enumeration (BloodHound)

For visualising nested group membership and abusable group delegation (often far larger than direct membership suggests), collect the domain with BloodHound. See [[AD Automatic Enumeration (BloodHound)]].

```bash
$ sudo bloodhound-python -u 'USERNAME' -p 'PASSWORD' -ns DC_IP -d DOMAIN_FQDN -c all
```

> [!Note]
> **Unrolled Members** in BloodHound shows the real number of users that effectively belong to a group through nested memberships, no matter how many layers deep. This is usually the value that matters for privilege escalation.

> [!Important] High-value built-in groups to look for
> - `Domain Admins` / `Enterprise Admins` / `Administrators` — full control of the domain / forest.
> - `Account Operators` — can manage non-protected accounts and add members to many groups.
> - `Backup Operators` — can read/write any file via backup privileges (path to DC compromise).
> - `Server Operators` — can manage services on DCs.
> - `DnsAdmins` — can load an arbitrary DLL into the DNS service running as SYSTEM.
> - `Remote Management Users` — WinRM access (see [[WinRM]]).

# Related notes
- [[AD Enumerating Users]]
- [[AD Enumeration with PowerView]]
- [[AD Automatic Enumeration (BloodHound)]]
- [[AD ACL Enumeration and Abuse]]