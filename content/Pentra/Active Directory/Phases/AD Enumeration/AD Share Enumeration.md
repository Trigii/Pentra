---
title: AD Share Enumeration
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
- Using CME:
```bash
$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD --shares # retrieve a list of shares on the target and the level of access our user has

$ sudo crackmapexec smb TARGET_IP_OR_FQDN -u USERNAME -p PASSWORD -M spider_plus --share 'SHARE_NAME' (list all readable files from a share)
NOTE: results are stored at /tmp/cme_spider_plus/<ip of host>.
Check the results:
$ head -n 10 /tmp/cme_spider_plus/TARGET_IP.json
```

- SMBMap:
```bash
$ smbmap -u USERNAME -p PASSWORD -d DOMAIN_FQDN -H TARGET_IP_OR_FQDN # list shares and check access)

$ smbmap -u USERNAME -p PASSWORD -d DOMAIN_FQDN -H TARGET_IP_OR_FQDN -R 'SHARE_NAME' --dir-only # list recursively only dirs inside the share
```

- smbclient (interactive access to a share):
```bash
$ smbclient -L //TARGET_IP -U DOMAIN/USERNAME # list shares

$ smbclient //TARGET_IP/SHARE_NAME -U DOMAIN/USERNAME # connect to a share
smb: \> ls          # list files
smb: \> get FILE    # download a file
smb: \> prompt off; recurse on; mget *   # download everything recursively
```

- Null / guest session (unauthenticated, worth trying first):
```bash
$ smbclient -L //TARGET_IP -N            # null session (no credentials)

$ smbmap -u '' -p '' -H TARGET_IP        # anonymous share listing

$ smbmap -u 'guest' -p '' -H TARGET_IP   # guest access
```

- rpcclient / enum4linux-ng (broader SMB + RID enumeration):
```bash
$ rpcclient -U "" -N TARGET_IP           # null session
rpcclient $> netshareenumall             # enumerate shares
rpcclient $> enumdomusers                # enumerate domain users

$ enum4linux-ng -A TARGET_IP             # automated: shares, users, groups, password policy
```

> [!Info]
> Recommended for enumerating SMB shares. Always try a **null/guest session** first — misconfigured shares often expose credentials, config files or scripts without any authentication.

---

### Related notes
- [[SMB]] — SMB service fundamentals and general enumeration.
- [[AD Enumeration - Credentialed - From Linux]] — credentialed enumeration workflow from Kali.
- [[AD Enumerating Users]] — turning share/RID findings into a user list.