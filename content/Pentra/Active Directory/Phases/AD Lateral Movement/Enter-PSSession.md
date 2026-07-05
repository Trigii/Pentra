---
title: Enter-PSSession
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 

> [!Requirements]
> Valid user credentials and remote management permissions (the target user must be a member of the `Remote Management Users` group, or a local administrator on the target).

`Enter-PSSession` opens an interactive PowerShell remoting session over WinRM (PSRemoting), giving us a remote shell on the target host. It relies on the same underlying transport as [[WinRM]] (ports 5985/HTTP and 5986/HTTPS) and is a common lateral movement technique once we hold valid domain credentials.

- If we are authenticating as a different user than the current:
```powershell
PS> $user = "DOMAIN_NAME\USER"  
PS> $Password = ConvertTo-SecureString "PASSWORD " -AsPlainText -Force  
PS> $credentials = New-Object System.Management.Automation.PSCredential ($user, $Password)

PS C:\> Enter-PSSession -ComputerName "TARGET_FQDN" -Credential $credentials
```

- If we are authenticating as the current user:
```powershell
PS C:\> Enter-PSSession -ComputerName "TARGET-FQDN"
```

- We can also run a single command remotely without opening a full interactive session using `Invoke-Command`:
```powershell
PS C:\> Invoke-Command -ComputerName "TARGET_FQDN" -Credential $credentials -ScriptBlock { whoami; hostname }
```

- Exit the remote session when finished:
```powershell
[TARGET-FQDN]: PS C:\> exit
```

> [!Note]
> If credentials are reused as a hash rather than a plaintext password, prefer a Pass-the-Hash capable tool such as `evil-winrm` (see [[WinRM]]) or [[Pass the Hash (PtH)]], since `Enter-PSSession` itself requires a plaintext password (or a Kerberos ticket via [[Pass the Ticket (PtT)]]).

> [!Tip]
> Other lateral movement options over different protocols include [[WMI]], [[RDP]] and [[Impacket]] (psexec/wmiexec/atexec).
