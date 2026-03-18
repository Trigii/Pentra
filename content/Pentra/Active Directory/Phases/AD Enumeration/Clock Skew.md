---
title: Clock Skew
draft: false
tags:
---
 
1. Enumerate the DC clock:
```bash
$ nmap -sV --script smb2-time -p 445 DC_IP
```

2. Kerberos rejects anything beyond ±5 minutes. Must sync before proceeding:
```

```
