---
title: AD Password Policy Enumeration
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
# Enumerating the Password Policy

> [!Requirements]
> - SMB NULL session
> - LDAP anonymous bind
> - Set of valid credentials (domain credentials are valid)

# Linux

## Enumerating the Password Policy - from Linux - Credentialed

```shell
crackmapexec smb TARGET_IP -u USER -p PASSWORD --pass-pol
```

## Enumerating the Password Policy - from Linux - SMB NULL Sessions

- Connect via SMB using a NULL session:
```bash
$ rpcclient -U "" -N TARGET
```

- Query domain info and confirm NULL access:
```bash
rpcclient $> querydominfo
```

- Query password policy:
```shell
rpcclient $> getdompwinfo
```

- Using enum4linux:
```shell
$ enum4linux -P TARGET

or

$ enum4linux-ng -P TARGET -oA OUTPUT

Parameters:
-oA: output in all formats for prettier results (YAML and JSON)
```

---
# Windows

## Enumerating the Password Policy - from Windows

- Using LOTL tools:
```powershell
C:\htb> net accounts # enumerate password policy for current user
```

- Using PowerView:
```powershell
PS C:\htb> import-module .\PowerView.ps1
PS C:\htb> Get-DomainPolicy
```

## Enumerating the Password Policy - from Windows - SMB NULL Sessions

- Establishing a NULL session:
```powershell
C:\> net use \\HOST_FQDN\ipc$ "" /u:""
The command completed successfully.

+ Error: account is disabled +
C:\> net use \\HOST_FQDN\ipc$ "" /u:guest
System error 1331 has occurred.
This user can't sign in because this account is currently disabled.

+ Error: Password is Incorrect +
C:\> net use \\HOST_FQDN\ipc$ "password" /u:guest
System error 1326 has occurred.

The user name or password is incorrect.

+ Error: Account is locked out (Password Policy) +
C:\> net use \\DC01\ipc$ "password" /u:guest
System error 1909 has occurred.

The referenced account is currently locked out and may not be logged on to.
```

## Enumerating the Password Policy - from Linux - LDAP Anonymous Bind

- Using ldapsearch (we can also use `windapsearch.py`, `ldapsearch`, `ad-ldapdomaindump.py`, etc.)
```shell
$ ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```
> [!Note]
> The b "DC=..." parameter is used to specify the domain FQDN. In the example above, the domain FQDN is `INLANEFREIGHT.LOCAL` so we have to specify twice the DC=

