---
title: Clock Skew
draft: false
tags:
---
 
1. Enumerate the DC clock:
```bash
$ nmap -sV --script smb2-time -p 445 DC_IP
```

2. Kerberos rejects anything beyond ±5 minutes (the default `MaxClockSkew`). A skewed clock produces errors such as `KRB_AP_ERR_SKEW` or impacket's `Kerberos SessionError: KRB_AP_ERR_SKEW (clock skew too great)`. Must sync the attacker clock to the DC before proceeding:

```bash
# Option A — one-off hard sync to the DC (requires root). Stop NTP first so it doesn't fight you:
$ sudo systemctl stop systemd-timesyncd 2>/dev/null
$ sudo ntpdate <DC_IP>           # from ntpdate / ntpsec-ntpdate
# or, if ntpdate is unavailable:
$ sudo rdate -n <DC_IP>

# Option B — sync against the DC by hostname (resolves via the domain DNS = the DC):
$ sudo ntpdate dc01.domain.local

# Verify the offset is now within tolerance:
$ ntpdate -q <DC_IP>             # query only, shows the remaining offset
$ date
```

> [!Tip]
> Newer Impacket and NetExec releases accept the `-k` flag together with an automatic skew correction, and many tools honour the `faketime` wrapper to spoof the clock for a single command without touching the system clock:
> ```bash
> $ faketime "$(ntpdate -q <DC_IP> | awk '{print $1, $2}')" impacket-GetUserSPNs ...
> ```

> [!Note]
> Always fix clock skew **before** any Kerberos-based attack: [[Kerberoasting]], [[AS-REP Roasting]], [[Pass the Ticket (PtT)]] and [[Pass the Hash (PtH)]] over Kerberos all fail with `KRB_AP_ERR_SKEW` when the clock drifts. See also [[AD Initial Foothold]].
