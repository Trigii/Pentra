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

> [!Info]
> Recommended for enumerating SMB shares