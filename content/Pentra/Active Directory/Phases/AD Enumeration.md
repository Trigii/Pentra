---
title: AD Enumeration
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
# Password Spraying
- Identify Domain Users:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1
PS> Get-DomainUser | Select-Object -ExpandProperty cn | Out-File users.txt (get domain users and save them on a wordlist)
PS> Get-ADUser -Filter *
```

> [!Note]
> We use Select-Object instead of select because it prints the users without any column name

- Perform the password spray:
```powershell
PS> . .\DomainPasswordSpray.ps1
PS> Invoke-DomainPasswordSpray -UserList PATH_TO_USERS_WORDLIST -Password PASSWORD -Verbose
Parameters:
PASSWORD: common password (for example 123456 or password...)
```

---
# AD Enumeration with BloodHound
- Run the collector:
```powershell
+ RUN THE COLLECTOR +
PS> cd C:\Tools\BloodHound\BloodHound\resources\app\Collectors
PS> powershell -ep bypass
PS> . .\SharpHound.ps1
PS> Invoke-Bloodhound -CollectionMethod All
```

- Run Bloodhound:
```powershell
1. Run the binary under BloodHound/BloodHound
User: neo4j
Password: Password@123
2. Click on upload data -> BloodHound/BloodHound/resources/app/Collectors/FOLDER.zip
3. Database Info -> Refresh Database Stats
4. Navigate to analysis and execute queries
```

>[!Note]
>**DCSync** is an attack technique that allows an attacker to simulate the behaviour of a domain controller and extract password data through domain replication. The primary purpose of this attack is often to obtain the KRBTGT hash, which can be a prelude for launching a [Golden Ticket attack](https://blog.quest.com/golden-ticket-attacks-how-they-work-and-how-to-defend-against-them/). It is implemented as a command in tools like Mimikatz, leveraging the Directory Replication Service Remote Protocol (MS-DRSR) to mimic a domain controller's behavior and request replication from other domain controllers.

Group members:
- **Direct Members:** The number of principals that have been directly added to this group.
- **Unrolled Members:** The actual number of users that effectively belong to this group, no matter how many layers of nested group membership that goes.
- **Foreign Members:** The number of users from other domains that belong to this group.

Group membership (from a user):
- **First Degree Group Memberships:** AD security groups the user is directly added to.
- **Unrolled Group Membership:** Groups can be added to groups, and those group nestings can grant admin rights, control of AD objects, and other privileges to many more users than intended. These are the groups that this user effectively belongs to, because the groups the user explicitly belongs to have been added to those groups.
- **Foreign Group Membership:** Groups in other Active Directory domains this user belongs to.

> [!NOTE] 
> OUs with GPOs can contain other OUs inside, and they will be affected too by the GPOs

Outbound Object Control:
- **First Degree Object Control:** The number of objects in AD where this user is listed as the IdentityReference on an abusable ACE. In other words, the number of objects in Active Directory that this user can take control of without relying on security group delegation.
- **Group Delegated Object Control:** The number of objects in AD where this user has control via security group delegation, regardless of how deep those group nestings may go.
- **Transitive Object Control:** The number of objects this user can gain control of by performing ACL-only based attacks in Active Directory. In other words, the maximum number of objects the user can gain control of without needing to pivot to any other system in the network, just by manipulating objects in the directory.

---
# AD Enumeration with PowerView
- Load PowerView:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1
```

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

- Domain Enumeration:
```powershell
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

- User Enumeration:
```powershell
PS> Get-DomainUser (get all users from the domain; not formatted correctly)
PS> Get-DomainUser | select samaccountname, objectsid (get all users; formatted users)
PS> Get-DomainUser -Identity USER (display a particular domain user)
PS> Get-DomainUser -Identity USER -Properties DisplayName,MemberOf,objectsid,useraccountcontrol | FormatList (display only specific properties or fields for a specific user)
PS> Find-LocalAdminAccess (not completed. Identify machines where a specific user has local admin rights)
```

- Computer enumeration:
```powershell 
PS> Get-NetComputer (display all computers in the current domain)
PS> Get-NetComputer | select name (filter specific properties)
PS> Get-NetComputer | select name,cn,operatingsystem
PS> Get-NetComputer -Domain FQDN_DOMAIN | select cn, operatingsystem (list specific properties of computers in a different domain)
```

- Group Enumeration:
```powershell
PS> Get-NetGroup (display all groups in the current domain)
PS> Get-NetGroup | select name (filter groups)
PS> Get-NetGroup 'GROUP' (get info about a specific group. ex: Domain Admins)
PS> Get-NetGroupMember "GROUP" | select MemberName (filter users that belong to a group. ex: Domain Admins)
PS> Get-NetGroup -userName "USER" | select name (display groups that the USER belongs to)
```

- Share Enumeration:
```powershell
PS> Find-DomainShare -ComputerName FQDN_COMPUTER_NAME -verbose (list shares from a system on the current domain. Ex: prod.research.SECURITY.local -> DC FQDN host)
PS> Find-DomainShare -ComputerName FQDN_COMPUTER_NAME -CheckShareAccess -verbose (enumerate all shares that the current user has read access to)
PS> Get-NetShare (list active shares on the current host)
```

> [!NOTE] 
> The **SYSVOL** share is an important share as it is responsible for storing and replicating important domain-related data and files, such as Group Policy Objects (GPOs) and logon scripts. The SYSVOL share is automatically created on each domain controller in an Active Directory domain and is shared by default. It serves as a central repository for GPOs, which are used to manage security policies, software deployment, and other configuration settings across the domain. All the domain computers access this share to check the domain policies.

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
PS> Get-DomainObjectAcl -Identity WAYNE_GIBBS -ResolveGUIDs (enumerate ACLs associated with a specific object)

PS> Find-InterestingDomainAcl -ResolveGUIDs (list interesting ACEs)
PS> Find-InterestingDomainAcl -ResolveGUIDs | select IdentityReferenceName, ObjectDN, ActiveDirectoryRights (search interesting ACEs and filter the output)
PS> Get-ObjectAcl -SamAccountName WAYNE_GIBBS -ResolveGUIDs | ? {$_.ActiveDirectoryRights -eq "GenericAll"} (Search for a specific Active Directory right associated with the specified object)
Note that **GenericAll** is a highly permissive right that typically provides full control over the object.
```

- Kerberoastable and AS-REP roastable accounts enumeration:
```powershell
PS> Get-NetUser -SPN | select samaccountname, serviceprincipalname (find kerberoastable accounts: Identify user accounts with non-null Service Principal Name (SPN))
PS> Get-NetUser -PreauthNotRequired | select samaccountname, useraccountcontrol (list as-rep roastable accounts: Identify user accounts that have Pre-Authentication disabled)
```
