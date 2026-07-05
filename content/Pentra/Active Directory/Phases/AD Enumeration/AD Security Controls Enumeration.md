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

- Enumerate the Windows Firewall state and rules:
```powershell
PS C:\htb> Get-NetFirewallProfile | select Name, Enabled   (check if Domain/Private/Public profiles are on)

PS C:\htb> Get-NetFirewallRule | where {$_.Enabled -eq 'True' -and $_.Direction -eq 'Inbound'} | select DisplayName
```

- Enumerate installed AV/EDR products (via WMI):
```powershell
PS C:\htb> Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntiVirusProduct

PS C:\htb> Get-Service | where {$_.DisplayName -match 'defender|sentinel|crowdstrike|carbon|cylance'}
```

> [!Note]
> Knowing which controls are active tells us **how** to enumerate. If Windows Defender or Constrained Language Mode is active, PowerView will likely be blocked — fall back to living-off-the-land techniques in [[AD Enumeration (LOTL)]] instead of [[AD Enumeration with PowerView]] or [[AD Automatic Enumeration (BloodHound)]].

> [!Tip]
> Related: this enumeration is part of the broader [[Active Directory Penetration Testing]] workflow. Bypassing these controls (Defender, AMSI, AppLocker) is covered under obfuscation — see [[Obfuscation]].
