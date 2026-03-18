---
title: AD Enumerating Groups
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
# LOTL Tools

- Net:
```powershell
C:\> net group /domain # enumerate ALL domain groups

C:\> net group "GROUP" /domain # enumerate members of a group
```

- Crackmapexec:
```bash
$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --groups # retrieve a list of all domain groups
```

- [Windapsearch](https://github.com/ropnop/windapsearch):
```bash
$ python3 windapsearch.py --dc-ip DC_IP -u USERNAME@DOMAIN_FQDN -p PASSWORD --da # enumerate domain admins group members
```