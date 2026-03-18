---
title: AD Enumeration - Credentialed - From Linux
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
> [!Requirements]
> Valid Domain credentials at any permission level:
> - Domain User's cleartext password
> - NTLM password hash
> - SYSTEM access on a domain-joined host

- CrackMapExec:
```bash
$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --users (retrieve a list of all domain users)

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --groups (retrieve a list of all domain groups)

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --loggedon-users (retrieve a list of all users that are currently logged in)

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --shares (retrieve a list of shares on the target and the level of access our user has)

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD -M spider_plus --share 'SHARE_NAME' (list all readable files from a share)
NOTE: results are stored at /tmp/cme_spider_plus/<ip of host>.
Check the results:
$ head -n 10 /tmp/cme_spider_plus/TARGET_IP.json
```

- SMBMap:
```bash
$ smbmap -u USERNAME -p PASSWORD -d DOMAIN_FQDN -H TARGET_IP_OR_FQDN (list shares and check access)

$ smbmap -u USERNAME -p PASSWORD -d DOMAIN_FQDN -H TARGET_IP_OR_FQDN -R 'SHARE_NAME' --dir-only (list recursively only dirs inside the share)
```

> [!Info]
> Recommended for enumerating SMB shares

- rpcclient:
```bash
$ rpcclient -U "" -N TARGET (unauthenticated login: NULL session)

$ rpcclient -U "AD_USER%PASSWORD" TARGET

rpcclient $> enumdomusers (enumerate all users and RIDs)

rpcclient $> queryuser 0xRID (enumerate user by RID)
```

> [!Note]
> Relative Identifier (RID): unique identifier (represented in hexadecimal format) utilized by Windows to track and identify objects.
> 
> Each Domain has an SID associated. When an object is created within a domain, the SID of the domain will be combined with the RID of the object to make a unique value that represents the object.
> 
> **Administrator RID** = `0x1f4`

- Impacket toolkit:

> [!Requirements]
> - Credentials for a user with **local administrator privileges** on the host

```bash
$ psexec.py DOMAIN_FQDN/ADMIN_USER:'PASSWORD'@TARGET (access the host as SYSTEM user)

$ wmiexec.py DOMAIN_FQDN/ADMIN_USER:'PASSWORD'@TARGET (access the host as the local admin user we connected with)
```

- [Windapsearch](https://github.com/ropnop/windapsearch):
```shell
$ python3 windapsearch.py --dc-ip DC_IP -u USERNAME@DOMAIN_FQDN -p PASSWORD --da (enumerate domain admins group members)

$ python3 windapsearch.py --dc-ip DC_IP -u USERNAME@DOMAIN_FQDN -p PASSWORD -PU (find privileged users)
```
