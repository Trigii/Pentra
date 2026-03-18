---
title: AD Automatic Enumeration (BloodHound)
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
> [!Requirements]
> - Download the latest version of  [SharpHound](https://github.com/BloodHoundAD/SharpHound/releases)
> - Authenticate as a domain user from a Windows attack host positioned within the network (but not joined to the domain)
> - Or transfer the tool to a domain joined host

# From Linux
- Usage:
```shell
1. Run the collector:
$ sudo bloodhound-python -u 'USERNAME' -p 'PASSWORD' -ns TARGET_IP -d DOMAIN_FQDN -c all (start the collector and obtain all the metrics as possible)

2. Start the neo4j service to load the collected data:
$ sudo neo4j start

3. Zip all collected json files:
$ zip -r DOMAIN.zip *.json

4. Start bloodhound:
$ bloodhound

5. Click the `Upload Data` button on the right side of the window

6. Run queries or use [custom Cypher queries](https://hausec.com/2019/09/09/bloodhound-cypher-cheatsheet/)
```

- Alternative:
```bash
$ sudo netexec ldap DC_IP -u 'AD_USER' -p 'AD_PASSWORD' --bloodhound -d DOMAIN_FQDN --dns-server DC_IP
```

---
# From Windows

- Run the collector (on a domain-joined host):
```powershell
+ RUN THE COLLECTOR +
PS> cd C:\Tools\BloodHound\BloodHound\resources\app\Collectors
PS> powershell -ep bypass
PS> . .\SharpHound.ps1
PS> Invoke-Bloodhound -CollectionMethod All
PS> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\<USER>\Desktop\ -OutputPrefix "DOMAIN audit" # for better output

+ ALTERNATIVE +
PS> .\SharpHound.exe -c All --zipfilename NAME
```

- Run Bloodhound (on local windows attack machine):
```powershell
1. Start Neo4j database
$ sudo neo4j start
2. Run the bloodhound binary:
$ bloodhound # linux
PS C:\> BloodHound/BloodHound.exe # windows
User: neo4j
Password: Password@123
3. Click on upload data -> BloodHound/BloodHound/resources/app/Collectors/FOLDER.zip
4. Database Info -> Refresh Database Stats
5. Navigate to analysis and execute queries:

- Find Computers with Unsupported Operating Systems
- Find Computers where Domain Users are Local Admin (hosts where all users have local admin rights)
```

- Run BloodHound (on local linux attack machine):
```bash
$ bloodhound-python -d CURRENT_DOMAIN_FQDN -dc CURRENT_DC_HOSTNAME -c All -u CURRENT_DOMAIN_USER -p PASSWORD
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

> [!Note]
> Owned Principals means the objects in AD we are currently in control. Use it in queries like `Shortest Paths to Domain Admins from Owned Principals`. To mark an object as Owned Principal we have to search it, right click on it and mark object as owned.

Custom queries:
```bash
1. Enumerate computers in the domain:
MATCH (m:Computer) RETURN m

2. Enumerate users in the domain:
MATCH (m:User) RETURN m

3. Enumerate domain user active sessions in domain computers
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p

```

Recommended queries:
1. Kerberoastable Users -> hang fruit
2. AS-REP roastable Users -> hang fruit
3. Shortes paths from Owned Objects -> use every time we set a new owned object
4. Shortes paths to systems trusted from unconstrained delegation

