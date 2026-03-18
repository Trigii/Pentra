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
> Valid user credentials

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
