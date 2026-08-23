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

---

## Why the policy matters

The password policy is the single most important input before any online guessing attack. The two fields to read carefully are:

- **Lockout Threshold** — how many failed attempts before an account locks. If it is `0` (never), you can spray/brute force freely; if it is low (e.g. 3–5), you must throttle attempts to avoid locking accounts and alerting the blue team.
- **Lockout Observation / Reset Window** — how long to wait between spray rounds so failed attempts "age out" and don't accumulate toward the threshold.
- **Minimum Password Length / Complexity** — tells you how to build a realistic candidate wordlist (e.g. `Season+Year!`, `Company123!`).

> [!Tip]
> Read the policy **before** running [[AD Password Spraying]]. A safe spray does **one** password per user per observation window. Example: threshold of 5 with a 30-minute window → spray a single candidate, wait, then try the next. `netexec`/`crackmapexec smb ... --pass-pol` shows all of these values in one shot.

## Getting the policy with NetExec (modern CME successor)

```shell
$ netexec smb TARGET_IP -u USER -p PASSWORD --pass-pol
```

---

## Related notes

- [[AD Password Spraying]] — the direct consumer of this policy (respect the lockout threshold).
- [[AD Enumeration - Credentialed - From Linux]] / [[AD Enumeration - Credentialed - From Windows]] — broader credentialed enumeration once you hold valid creds.
- [[SMB]] — NULL sessions and `rpcclient`/`enum4linux` transport used above.
- [[LDAP]] — anonymous bind used to read `pwdHistoryLength` and related attributes.
- [[Active Directory Penetration Testing]] — where password-policy enumeration fits in the overall workflow.

