---
title: AD Enumeration with PowerView
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
- Load PowerView:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1 \\ Import-Module .\PowerView.ps1
```

- Credential Object:
```powershell
PS C:\> $SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
PS C:\> $Cred = New-Object System.Management.Automation.PSCredential('DOMAIN\USER', $SecPassword) 
PS C:\> Find-LocalAdminAccess -Domain DOMAIN -Credential $Cred
```

# Local Enumeration

- Enumerate local users and groups:
```powershell
PS> whoami (current user)
PS> hostname (current host)
PS> whoami /priv (current user privileges)
PS> whoami /all (enumerate everything about the current user)
PS> Get-LocalUser (list local users)
PS> Get-LocalUser | Filter Name,Enabled,LastLogon
PS> net accounts (list account policy settings)
PS> net user administrator (get details about a local user)

PS> Get-LocalGroup (list local groups)
PS> Get-LocalGroup | ft Name
PS> Get-LocalGroupMember Administrators (get users that belong to a group)
Note: Domain Admins group is a group created automaticaly when a domain is set up and it has the highest privileges in the domain. For each system, its part of the Administrator group.
PS> ipconfig /all (display network interfaces, IPs, and DNS)
PS> Get-NetIPConfiguration | ft InterfaceAlias,InterfaceDescription,IPv4Address (enumerate net info better -> IP address)
PS> Get-DnsClientServerAddress -AddressFamily IPv4 (enumerate net info again -> DNS)
```

---
# Domain Enumeration

- Domain Enumeration:
```powershell
PS> Get-NetDomain (basic info about the domain)
PS> Get-Domain (get domain info, including parent domain)
PS> Get-Domain -Domain FQDN_DOMAIN (get info about the specified domain. For example: parent domain)
PS> Get-DomainSID (get curren domain SID)
PS> Get-DomainPolicy (get domain policy about system access or kerberos)
PS> (Get-DomainPolicy)."SystemAccess" (get domain policy about system access)
PS> (Get-DomainPolicy)."kerberospolicy" (get domain policy about kerberos)
```

- DC Enumeration:
```powershell
PS> Get-DomainController (get info about the DC of the current domain)
PS> Get-DomainController -Domain FQDN_DOMAIN (get info about DC of a specific domain)
```

- Domain User Enumeration:
```powershell
PS> Get-NetUser (get a list of all users in the domain)
PS> Get-NetUser | select <ATRIBUTE> (get a list of all users in the domain and filter by an attribute: recommended "cn" for printing only usernames)
PS> Get-NetUser | select cn,pwdlastset,lastlogon (enumerate last password change from domain users and last logins)
PS> Get-DomainUser (get all users from the domain; not formatted correctly)
PS> Get-DomainUser | select samaccountname, objectsid (get all users; formatted users)
PS> Get-DomainUser -Identity USER (display a particular domain user)
PS> Get-DomainUser -Identity USER -Properties DisplayName,MemberOf,objectsid,useraccountcontrol | FormatList (display only specific properties or fields for a specific user)
PS> Get-DomainUser -Identity USER -Domain DOMAIN_FQDN | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol (full user enumeration)
PS> Find-LocalAdminAccess [-Computername] [-Credentials](Identify machines where a specific user has local admin rights. If we dont specify any parameter, it will attempt to enumerate all computers in the domain where we have admin rights for the current user we are loged on)
PS> Get-DomainUser * | Select-Object samaccountname,description |Where-Object {$_.Description -ne $null} (enumerate users with associated descriptions to look for plaintext passwords)
PS> Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol (enumerate users with the PASSWD_NOTREQD set)

Important:
PS> Test-AdminAccess -ComputerName ACADEMY-EA-MS01 (test if the current user is admin on a specific host - current or remote)
```

> [!Note]
> Admin rights on computers allow users to gain RCE in several ways: RDP (recommended), WinRM, WMI.... They can also impersonate other users.

- Enumerate users are loged in to a specified computer: 
```powershell
PS C:\> Get-NetSession -ComputerName HOST_CN -Verbose 
```

Useful for extracting credentials from memory on a specific computer (we need admin access)

> [!Requirements]
> The permissions required to enumerate sessions with _NetSessionEnum_ are defined in the **SrvsvcSessionInfo** registry key, which is located in the **HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity** hive.
> We can enumerate it with `PS C:\> Get-Acl -Path HKLM:SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity\ | fl`
> This command typically works only on older systems.

Alternative tool:
```powershell
PS C:\> .\PsLoggedon.exe \\COMPUTER_CN
```

> [!Requirements]
> _Remote Registry_ service enabled. If we get no users connected to a computer, it might be a false positive.

- Computer enumeration:
```powershell 
PS> Get-NetComputer (display all computers in the current domain: OS, dns hostname)
PS> Get-NetComputer | select name (filter specific properties)
PS> Get-NetComputer | select name,cn,operatingsystem,operatingsystemversion,dnshostname
PS> Get-NetComputer -Domain FQDN_DOMAIN | select cn, operatingsystem (list specific properties of computers in a different domain)
```

- Domain Group Enumeration:
```powershell
PS> Get-NetGroup (display all groups in the current domain)
PS> Get-NetGroup | select name (display only group names)
PS> Get-NetGroup | select cn (display only group names)
PS> Get-NetGroup 'GROUP' (get info about a specific group. ex: Domain Admins)
PS> Get-NetGroupMember "GROUP" | select MemberName (filter users that belong to a group. ex: Domain Admins)
PS> Get-NetGroup "Sales Department" | select member (filter users that belong to a group; alternative)
PS> Get-NetGroup -userName "USER" | select name (display groups that the USER belongs to)

Important:
PS> Get-DomainGroupMember -Identity "Domain Admins" -Recurse (enumerate users whom belong to a group. Recurse flag will enumerate groups that are part of the target group, and therefore, inherit the permissions)

Additional commands:
C:\> net group "GROUP" USER /add /domain # add user to a group (ACL Write permissions required)
```

- Domain Services Enumeration:

Service Principal Names, also known as Unique Service Instance Identifiers, associates a service to a specific service account (LocalSystem, LocalService, NetworkService) in Active Directory.

We can obtain the IP address and port number of applications running on servers integrated with AD by simply enumerating all SPNs in the domain:
```powershell
+ ENUMERATE SPN FOR SPECIFIC USER +
C:\> setspn -L DOMAIN_USER # enumerate SPNs for a specific domain user account (for example iis_service)

+ ENUMERATE ALL SPNs +
PS C:\> Get-NetUser -SPN | select samaccountname,serviceprincipalname # (powerview required)
```

> [!Note]
> We can resolve the service instances where the services are running with:
> `C:\> nslookup.exe INSTANCE_FQDN`

- Share Enumeration:
```powershell
PS> Find-DomainShare # list all shares in the domain
PS> Find-DomainShare -CheckShareAccess # list all domain shares in the domain we can access (careful because we might be able to access other shares with a different user)
PS> Find-DomainShare -ComputerName FQDN_COMPUTER_NAME -verbose (list shares from a system on the current domain. Ex: prod.research.SECURITY.local -> DC FQDN host)
PS> Find-DomainShare -ComputerName FQDN_COMPUTER_NAME -CheckShareAccess -verbose (enumerate all shares that the current user has read access to)
PS> Get-NetShare (list active shares on the current host)

SYSVOL Enumeration:
PS C:\> ls \\HOSTNAME\SYSVOL\DOMAIN_FQDN\FOLDER_PATH # look for passwords in the SYSVOL share. If we find one, we can perform a password spraying attack

PS C:\> ls \\DC_FQDN\sysvol\DOMAIN_FQDN\
Example:
PS C:\> ls \\dc1.corp.com\sysvol\corp.com\Polices
Important: there is a folder called Polices that contains interesting information such as old polices with backup files that might contain passwords. This passwords are encrypted with MD5 and can be decrypted using the private key that has been posted on MSDN:
PS C:\> gpp-decrypt "ENCRYPTED_PASSWORD"
```

> [!NOTE] 
> The **SYSVOL** share is an important share as it is responsible for storing and replicating important domain-related data and files, such as Group Policy Objects (GPOs) and logon scripts. The SYSVOL share is automatically created on each domain controller in an Active Directory domain and is shared by default. It serves as a central repository for GPOs, which are used to manage security policies, software deployment, and other configuration settings across the domain. All the domain computers access this share to check the domain policies.
> 
> By default, the **SYSVOL** folder is mapped to **%SystemRoot%\SYSVOL\Sysvol\domain-name** on the domain controller and every domain user has access to it.

- GPO and OUs Enumeration:
```powershell
PS> Get-NetGPO (enumerate all GPOs in the current domain)
PS> Get-NetGPO | select displayname (filter GPOs by name)
PS> Get-NetOU (list OUs in the current domain)
PS> Get-NetOU | select name, distinguishedname (filter OUs by specific properties)
```

- Domain and Forest trust enumeration:
```powershell
PS> Get-NetDomainTrust (enumerate all domain trusts for the current domain)
PS> Get-NetForest (get details of current forest)
PS> Get-NetForestTrust (get trusts of the current forest)
PS> Get-NetForestTrust -Forest FQDN_FOREST (get trusts of a different forest.)
PS> Get-NetForest -Forest FQDN_FOREST (get details of a different forest. Note: we need trust from the other forest to be able to enumerate it)
PS> Get-NetForestDomain (get all domains in the current forest)
PS> Get-NetForestDomain -Forest FQDN_FOREST (get all domains in a different FOREST. Note: we need trust from the other domain)
PS> Get-DomainTrustMapping (enumerate all the trusts)
```

- ACLs Enumeration
```powershell
PS> Get-ObjectAcl -SamAccountName "Users" -ResolveGUIDs (enumerate ACLs associated with a specific object)
PS> Get-DomainObjectAcl -Identity "OBJECT" -ResolveGUIDs (enumerate ACLs associated with a specific object)

PS> Find-InterestingDomainAcl -ResolveGUIDs (list interesting ACEs)
PS> Find-InterestingDomainAcl -ResolveGUIDs | select IdentityReferenceName, ObjectDN, ActiveDirectoryRights (search interesting ACEs and filter the output)
PS> Get-ObjectAcl -Identity "USERNAME" (obtain ACEs for a specific user)

PS> Get-ObjectAcl -SamAccountName "OBJECT" -ResolveGUIDs | ? {$_.ActiveDirectoryRights -eq "GenericAll"} (Search for a specific Active Directory right associated with the specified object)
Note that **GenericAll** is a highly permissive right that typically provides full control over the object.

Alternative: search for a specific AD right associated with a specific Object:
PS C:\> Get-ObjectAcl -Identity "OBJECT" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights

Additional commands:
PS C:\> Convert-SidToName SID (convert an SID to name)
```

> [!Note]
> - If we have `GenericAll` or `GenericWrite` or similar over a user, we can change his password using:
> `$ net user /domain USER NEW_PASSWORD` -> `$runas.exe /user:DOMAIN\USER cmd` -> `PS> Find-LocalAdminAccess`
> 
> - Over a group, we can add a user we have control to that group and gain the privileges of the group:
> `$ net group "GROUP" USER /add /domain`

Permissions:
```
GenericAll: Full permissions on object
GenericWrite: Edit certain attributes on the object
WriteOwner: Change ownership of the object
WriteDACL: Edit ACE's applied to object
AllExtendedRights: Change password, reset password, etc.
ForceChangePassword: Password change for object
Self (Self-Membership): Add ourselves to for example a group
```

- Test local admin access:
```powershell
PS> Test-AdminAccess -ComputerName ACADEMY-EA-MS01 (test if the current user is admin on a domain host)
```

- Kerberoastable and AS-REP roastable accounts enumeration:
```powershell
+ KERBEROASTABLE +
PS> Get-NetUser -SPN | select samaccountname, serviceprincipalname (find kerberoastable accounts: Identify user accounts with non-null Service Principal Name (SPN) - or SPN attribute set)

alternative:

PS> Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName

+ AS-REP ROASTABLE +
PS> Get-NetUser -PreauthNotRequired | select samaccountname, useraccountcontrol (list as-rep roastable accounts: Identify user accounts that have Pre-Authentication disabled)
```

> [!Note]
> As an alternative, SharpView can be used for AD enumeration just like PowerView:
> - Enumerate info about a user: `PS> .\SharpView.exe Get-DomainUser -Identity forend`
