---
title: Active Directory Penetration Testing
draft: false
tags:
  - windows
  - active-directory
---
 
# Active Directory
Centralized framework for managing a large number of assets like systems, users, groups... and the policies.

**Use cases**
- User authentication and authorization: AD serves as a centralized authentication and authorization mechanism. Users can login to their computer or other network resources in the domain using their AD credentials. The access permissions are manages through roles and groups.
- Resource management: centralized management of network resources such as computers, printers, apps, shared folders...
- Group Policy management: security policies (password policy, ...)
- Directory Services: provides a hierarchical structure for organizing the network (domains, trees, forests...)

**Components**
- Domain: logical grouping of network objects (users, computers, resources...) that share a common directory database and security policies.
- Domain Controller: manage access to the resources in a domain. Holds an updated replica of the AD database and authenticates user logins.
- Tree: hierarchy of domains in AD (example.com -> eu.example.com and us.example.com -> ...) (creates trust between domains)
- Forest: collection of domain trees that share a global schema, configuration and global catalog (enables trust between all domains in the forest).
- **Group Policy Objects (GPOs)** are used to manage and control the behavior of user accounts and computer accounts within a domain. Each GPO contains a collection of settings and configurations that are applied to targeted users or computers in the domain. They can be linked to sites, domains, or OUs within the Active Directory hierarchy to define their scope and targeting. Some common use cases for GPOs include setting desktop wallpaper, managing software installations, restricting access to specific features or applications, defining security settings etc. GPOs can be abused for a variety of attacks, including privilege escalation, deploying backdoors, establishing persistence etc.
- **Organizational Units (OUs)** are containers within Active Directory that help organize and manage objects such as users, computers, groups, and other resources for easier management and application of policies. They provide a way to structure and delegate administrative control within the domain. By creating OUs, administrators can apply different GPOs to specific sets of users or computers, tailoring the policies to the unique needs of those groups. OUs can represent various aspects of an organization's structure, such as departments, geographical locations, or functional units. OUs can also be nested within each other to create a hierarchical organizational structure (for example using a OU for sorting or organize users, computer... in different departments in an organization)
- Global Catalog: distributed data repo that contains a replica of all objects in the forest. Facilitates cross-domain searches and enables users to locate resources across the entire AD forest.
- In an Active Directory environment, **trust** represents a relationship established between two domains or forests. This relationship enables users from one domain or forest to access resources located in the other domain or forest. **Domain Trust** enables authentication between domains within the same forest or across separate forests, facilitating resource sharing and collaboration. **Forest Trust** extends trust relationships beyond individual domains and encompasses the entire forest infrastructure, enabling authentication and resource access between domains in different forests.
	- Directional trust: unidirectional trust
	- Transitive trust: extends beyond a 2 domain trust to include other trusted domains (if a trusts b and a trusts c then c trusts b)

> [!NOTE] 
> A bidirectional Parent-child trust is automatically generated when a child domain is added to a parent domain

- **Access Control Lists (ACLs)**, are security mechanisms used in computer systems and networks to regulate access to resources. It consist of ACEs (Access Control Entries), which are the individual entries within an ACL that specify permissions for a particular user or group. Each ACE contains information about the security principal (user or group), the specific permissions granted or denied, and whether the ACE is inherited from a parent object or explicit to the current object. ACEs provide granular control over resource access by allowing administrators to define fine-tuned permissions for different entities. Attackers can abuse misconfigured or overly permissive ACLs in Active Directory to escalate privileges, gain unauthorized access, or manipulate permissions.
	- There are two types of ACLs that can be found within the security descriptor of a securable object. These are the **Discretionary ACL (DACL)** and the **System ACL (SACL)**
		- The **DACL** (often mentioned as the ACL) specifies the permissions (allowed or denied) granted to trustees (a user or group), on an object.
		- On the other hand, the **SACL** logs audit messages that track both successful and failed attempts to access the object.

> [!Note]
> Machines in the Domain Controllers group are also members of the Domain Computers group.

**Users, Groups and Computers**
Domain Users: security principals are entities (users, groups, computers,...) that can be assigned permissions to access resources in a Windows environment.
Groups:
- Security groups: manage access permissions to network resources. Users are added to groups and permissions are granted to these groups.
	- Domain admins: full admin control over the entire domain
	- Enterprise admin: full admin privileges within the AD forest
	- Server Operators: manage DC and member servers within the domanins
	- Backup Operators: be able to perform backup and restore operations on DC
	- Account Operators: manage users, groups and computers within the domain
	- Domain Users: individual accounts created within the AD domain
	- Domain Computers: to categorize computers joined to the AD domain
	- Domain Controllers: server that operates within the AD and responsible for authenticate users, authorizing access to resources, maintaining the AD DB...
- Distribution groups: for sending emails to a group of receipients.

```powershell
PS> net user (display local user accounts)
PS> net user /domain (display domain users)
PS> Get-ADUser -Filter * (get all AD users)
PS> Get-ADGroup -Filter * (get all groups)
PS> Get-ADOrganizationalUnit -Filter * (get all OUs)
Note: For OUs we have the LinkedGroupPolicyObjects (policies like password policy...)

For authentication as a domain user:
User: DOMAIN\USER
Password: password
```

**LDAP**
Protocol used to communicate with Active Directory.

According to [Microsoft's documentation](https://learn.microsoft.com/en-us/windows/win32/adsi/ldap-adspath?redirectedfrom=MSDN), we need a specific LDAP _ADsPath_ in order to communicate with the AD service. The LDAP path's prototype looks like this:
```
LDAP://HostName[:PortNumber][/DistinguishedName]
```

- Hostname: it can be a computer name, IP address or a domain name.

> [!Note]
> Setting the domain name as the hostname could potentially resolve to any of the DC. First, we have to identify the Primary DC, which is the one holding the `PdcRoleOwner` property

- Port number: set it when we are dealing with a domain that uses non-default ports.
- Distinguished Name: name that uniquely identifies an Object in AD. For example for a user called `Stephanie`, the DN would be:
```
CN=Stephanie,CN=Users,DC=corp,DC=com
```

- Script for obtaining full LDAP Path:
```powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name # get Primary DC
$DN = ([adsi]'').distinguishedName # get DN for the Domain 
$LDAP = "LDAP://$PDC/$DN" # get LDAP Path for PDC
$LDAP
```

**Kerberos**
Authentication protocol on AD
- KDC = AS+TGServer = DC -> authenticates the user and issues the TGT to the user
- TGS = provides the services and issues the ST to the user
**NTLM**
Used for backward compatibility (challenge-response with the NTLM hash of the users password)

# Pentesting Phases

1. [[AD External Recon and Enumeration Principles]]

2. [[AD Enumeration]]

3. [[AD Privilege Escalation]]

4. [[AD Lateral Movement]]

5. [[AD Persistence]]

6. [[AD ACL Enumeration and Abuse]]

7. [[AD Domain Trust Abuse]]