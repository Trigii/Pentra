---
title: RDP
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 

> [!Requirements]
> Control over a **local admin user** over target machine or a user that belongs to the `Remote Desktop Users` group

1. Enumerate members of the `Remote Desktp Users` group on a host:
```powershell
PS C:\> Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Desktop Users"
```

2. Enumerate with bloodhound if the Domain Users group (any user in the domain) has local admin rights or execution rights (RDP or WinRM) over any host.
```bash
1. Search for Domain Users@DOMAIN node
2. Go to Node Info
3. Check the Local Admin Rights and Execution Rights sections
```

3. If we gain control over a user through attacks like LLMNR or Kerberoasting, we can search for the username in Bloodhound to check the Execution Rights (directly or inherited):
```bash
1. Search for User@DOMAIN node
2. Go to Node Info
3. Check the Execution Rights sections
```

4. Use BloodHound queries:
```bash
1. Find Workstations where Domain Users can RDP
2. Find Servers where Domain Users can RDP
```

> [!Important]
> If we have RDP access, try to open a powershell.exe with admin privileges and enter the current user credentials. We might have more privileges if we run `whoami /priv`

**Tools for RDP**

- Linux: `xfreerdp`:
```bash
$ xfreerdp /u:USER /d:DOMAIN /v:TARGET_IP
```

- Windows: `mstsc.exe`

> [!Warning]
> Whenever we have valid AD credentials, we suggest using RDP as much as possible. If you use PowerShell Remoting and winrm to connect to a machine, you may no longer be able to run domain enumeration tools as you will experience the [Kerberos Double Hop](https://posts.slayerlabs.com/double-hop/) issue.

# Bonus

Enable RDP:
```powershell
PS C:\> reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v "AllowRemoteRPC" /t REG_SZ /d "1" /f
```

Add a user to the RDP group:
```powershell
PS C:\> net localgroup "Remote Desktop Users" <user> /add
```