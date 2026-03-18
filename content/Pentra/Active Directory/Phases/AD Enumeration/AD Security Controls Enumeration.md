---
title: AD Security Controls Enumeration
draft: false
tags:
  - windows
  - post-exploitation
  - recon
  - info-gathering
  - active
---
 
# Enumerate Security Contols

- Enumerate Windows Defender (blocks PowerView):
```powershell
PS C:\htb> Get-MpComputerStatus
*check RealTimeProtectionEnabled*
```

- Enumerate AppLocker (blocks apps like CMD or Powershell):
```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

- Enumerate Powershell Constraint Language Mode (blocks PS features):
```powershell-session
PS C:\htb> $ExecutionContext.SessionState.LanguageMode
```

- Enumerate LAPS (randomize admin passwords on hosts):
```powershell
PS C:\htb> Find-LAPSDelegatedGroups (enumerate groups delegaed to read LAPS passwords)

PS C:\htb> Find-AdmPwdExtendedRights (find users with "All Extended Rights" -> they can read LAPS passwords)

PS C:\htb> Get-LAPSComputers (enumerate hosts with LAPS enabled)
```
