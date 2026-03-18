---
title: AD Enumerating Users
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
Tactis:
- By leveraging an SMB NULL session to retrieve a complete list of domain users from the domain controller
- Utilizing an LDAP anonymous bind to query LDAP anonymously and pull down the domain user list
- Using a tool such as `Kerbrute` to validate users utilizing a word list from a source such as the [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames) GitHub repo, or gathered by using a tool such as [linkedin2username](https://github.com/initstring/linkedin2username) to create a list of potentially valid users
- Using a set of credentials from a Linux or Windows attack system either provided by our client or obtained through another means such as LLMNR/NBT-NS response poisoning using `Responder` or even a successful password spray using a smaller wordlist

> [!Requirements]
> - SMB NULL session on DC
> - LDAP anonymous bind on DC
> - Set of valid credentials for a domain user or SYSTEM access on a Windows Host

# LOTL Tools

> [!Requirements]
> - Valid domain credentials for a user.
> - Access to a domain joined host with the credentials.

- Net:
```powershell
C:\> net user /domain # enuemrate ALL domain users

C:\> net user USER /domain # enumerate a specific user
```

---
# PowerView

- Identify Domain Users and create a userlist:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1
PS> Get-DomainUser | Select-Object -ExpandProperty cn | Out-File users.txt (get domain users and save them on a wordlist)
PS> Get-ADUser -Filter *
```

> [!Note]
> We use Select-Object instead of select because it prints the users without any column name

---
# Enumerating users with SMB NULL Session
- Using enum4linux:
```shell
$ enum4linux -U DC_HOST  | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]" (this will print only the users list)
```

- Using rpcclient:
```bash
$ rpcclient -U "" -N DC_HOST
rpcclient $> enumdomusers 
```

- Using CME:
```bash
$ crackmapexec smb DC_HOST --users
```

---
# Enumerating users with LDAP Anonymous
- Using ldapsearch:
```bash
$ ldapsearch -v -x -b "DC=DOMAIN_NAME,DC=DOMAIN_NAME" -H "ldap://TARGET_IP" "(objectclass=user)"  | grep -E "sAMAccountName:" | cut -f2 -d" "
```

- Using windapsearch:
```shell
$ ./windapsearch.py --dc-ip DC_HOST -u "" -U

Parameters:
-u "": anonymous access
-U: retrieve only users
```

---
# Enumerating users with Kerbrute (recommended)

> [!Note]
> Kerbrute can be used to enumerate users in a domain if we have no access at all from our position in the internal networ

[Kerbrute](https://github.com/ropnop/kerbrute) can be a stealthier option for domain account enumeration. It takes advantage of the fact that Kerberos pre-authentication failures often will not trigger logs or alerts. The tool sends TGT requests to the domain controller without Kerberos Pre-Authentication to perform username enumeration. If the KDC responds with the error `PRINCIPAL UNKNOWN`, the username is invalid. Whenever the KDC prompts for Kerberos Pre-Authentication, this signals that the username exists, and the tool will mark it as valid.

- Compilation:
```shell
$ sudo git clone https://github.com/ropnop/kerbrute.git (download the repo)
$ make help (list compiling options)
$ sudo make all (compile for multiple platforms and architectures)
$ ls dist/ (list compiled binaries)
*move the binary to an executable path*
```

> [!Note]
> Depending on the attack machine we are going to use kerbrute, use the corresponding compiled binary

- Usage
```bash
$ kerbrute_linux_386 userenum -d DOMAIN_FQDN --dc DC_IP USER_WORDLIST -o valid_ad_users -t 100 --downgrade

Recommended user wordlist: /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
--downgrade: downgrade the hashing algorythm to RC4 to be able to crack the hashes offline
```

> [!Note]
> Kerbrute will automatically retrieve the AS-REP for any users that do not require the kerberos pre-authentication.
> For more info about AS-REP Roasting check [[AD Privilege Escalation]] 

---
# Enumerating users with valid credentials (recommended)

- Enumerate if a user has valid AD credentials using CME/enumerate if a user has admin privileges on a specific target:
```bash
$ sudo crackmapexec smb TARGET -u AD_USER -p PASSWORD
```

> [!Note]
> If `(Pawned!)` appears it means that the user has privileged access on that target.

- Using CME (we can use any of the other tools like enum4linux or rpcclient):
```shell
$ sudo crackmapexec smb TARGET -u AD_USER -p PASSWORD --users # retrieve a list of all domain users

Getting a list:
$ sudo crackmapexec smb DC_IP -u DOMAIN_USER -p DOMAIN_PASSWORD --users 2>/dev/null | grep -oP 'DOMAIN_CN\.DOMAIN_DN\\\K[^ ]+' > valid_users.txt

Example:
crackmapexec smb 172.16.7.3 -u AB920 -p weasal --users 2>/dev/null | grep -oP 'INLANEFREIGHT\.LOCAL\\\K[^ ]+' > valid_users.txt

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --loggedon-users # retrieve a list of all users that are currently logged in the target

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --shares # retrieve a list of shares on the target and the level of access our user has

+ KERBEROS AUTHENTICATION +
$ sudo crackmapexec smb TARGET_IP -u AD_USER -p PASSWORD -k # use kerberos authentication instead of NTLM

if we get the error KDC_ERR_WRONG_REALM we have to configure our linux to be able to know what Kerberos realm (domain) use and DC.

Execute:

```

- Kerberos authentication:

If we get an error using crackmapexec 

```bash
$ sudo crackmapexec smb TARGET_IP -u AD_USER -p PASSWORD -k # use kerberos authentication instead of NTLM
```

Reference: https://rudrasarkar.medium.com/story-of-understanding-web-vs-active-directory-domain-realm-kerberos-3a575353e1ad

If we get the error KDC_ERR_WRONG_REALM we have to configure our linux to be able to know what Kerberos realm (domain) use and DC:
```bash
sudo crackmapexec smb TARGET_IP -u AD_USER -p PASSWORD --generate-krb5-file krb5.conf -k
```

Export the file:
```
export KRB5_CONFIG=/Users/rudra/hackthebox/krb5.conf
```

- Using [Windapsearch](https://github.com/ropnop/windapsearch):
```bash
$ python3 windapsearch.py --dc-ip DC_IP -u USERNAME@DOMAIN_FQDN -p PASSWORD -PU # find privileged users
```

- Using rpcclient:
```bash
$ rpcclient -U "" -N TARGET # unauthenticated login: NULL session

rpcclient $> enumdomusers # enumerate all users and RIDs

rpcclient $> queryuser 0xRID # enumerate user by RID
```

> [!Note]
> Relative Identifier (RID): unique identifier (represented in hexadecimal format) utilized by Windows to track and identify objects.
> 
> Each Domain has an SID associated. When an object is created within a domain, the SID of the domain will be combined with the RID of the object to make a unique value that represents the object.
> 
> **Administrator RID** = `0x1f4

> [!Important]
> A `SYSTEM` account on a `domain-joined` host will be able to enumerate Active Directory by impersonating the computer account, which is essentially just another kind of domain user account and it will be treated like that. Having SYSTEM-level access within a domain environment is nearly equivalent to having a domain user account.
> 
> By gaining SYSTEM-level access on a domain-joined host, you will be able to perform actions such as, but not limited to:
> - Enumerate the domain using built-in tools or offensive tools such as BloodHound and PowerView.
> - Perform Kerberoasting / ASREPRoasting attacks within the same domain.
> - Run tools such as Inveigh to gather Net-NTLMv2 hashes or perform SMB relay attacks.
> - Perform token impersonation to hijack a privileged domain user account.
> - Carry out ACL attacks.