---
title: AD Linux Enumeration
draft: false
tags:
  - linux
  - active-directory
  - enumeration
  - post-exploitation
  - active
---
 
When we land on a **domain-joined Linux host**, enumerating how it is bound to the domain — and who logs into it — frequently hands us domain credentials or a lateral-movement path. Fix [[Clock Skew]] first if you intend to use any harvested Kerberos material, then reuse tickets/keytabs as described in [[Kerberos on Linux]].

# Enumerate auth attempts on Linux

If there is a domain joined host which is a Linux system, we might identify login attempts to the system:
```bash
cat /var/log/auth.log
```

Authentication types:
- **pam_unix**: attempts to authenticate via Linux (verifying local files to check the username/password like if it was a local linux user)
- **pam_sss**: attempts to authenticate via kerberos utilizing the provided credentials (verifying login agains DC)

If no encryption is being used (for example FTP auth with no encryption), and periodic login attempts are being made to the Linux system, we can extract the domain user credentials by running:
```bash
# for example for FTP (run from the Linux AD host):
sudo tcpdump -i any -A -nnv port 21 | grep -E "USER|PASS"

# SSH
sudo tcpdump -i any -nn 'tcp port 22 and (tcp[tcpflags] & tcp-syn != 0)'
```

# Confirm the host is domain-joined

Before anything else, verify the box is actually bound to Active Directory and identify the domain/DC:
```bash
# realmd / SSSD-managed join (most common on modern distros):
realm list                       # shows the joined realm, DC, and the login format

# The Kerberos and SSSD configs name the realm, KDC (=DC) and enrolled account:
cat /etc/krb5.conf               # [realms] / [domain_realm] -> DC hostname + FQDN
cat /etc/sssd/sssd.conf 2>/dev/null   # ad_domain, ad_server, ldap_uri (often root-only)

# Older winbind/Samba joins:
cat /etc/samba/smb.conf          # workgroup, realm, security = ADS
net ads info 2>/dev/null         # LDAP server, KDC, server time (also useful for [[Clock Skew]])
```

> [!Note]
> `/etc/krb5.keytab` (the computer-account keytab) usually exists on a properly joined host and can be abused directly — see [[Kerberos on Linux]] for stealing and using keytabs.

# Enumerate domain users and groups from the Linux host

Because AD identities are resolved through NSS, standard Linux tools return **domain** principals, not just local ones:
```bash
# Domain users/groups resolved via SSSD/winbind (backslash or @ format depending on the join):
getent passwd                    # local + cached/enumerable domain users
getent group  'Domain Admins'    # membership of a specific domain group
getent passwd 'DOMAIN\\jdoe'     # look up a single domain user

# Our own effective identity and group memberships (great for spotting privileged AD groups):
id
id 'DOMAIN\\administrator'
whoami /groups 2>/dev/null || id  # groups reveal Domain Admins / privileged SIDs
```

> [!Tip]
> Valid domain usernames harvested here feed straight into [[AD Password Spraying]] and Kerberos attacks. Cross-check with credentialed enumeration from the attack box in [[AD Enumeration - Credentialed - From Linux]].

# Check for AD-based privilege and stored secrets

```bash
# Sudo rights granted to domain groups (instant privesc if a group we're in is listed):
sudo -l                          # rules may reference %DOMAIN\ ad-group ALL=(ALL) ...
cat /etc/sudoers /etc/sudoers.d/* 2>/dev/null | grep -i '%\|@'

# Home directories of domain users (may hold notes, keys, ccache paths):
ls -la /home/                    # AD homes are often /home/DOMAIN/user or /home/user@domain

# Credentials sitting in bash history, config files and scripts:
grep -riE 'password|passwd|kinit|smbclient|-p ' /home /etc /opt /var/www 2>/dev/null | head
```

> [!Warning]
> Automated login attempts against the box (cron/keytab-driven scripts, monitoring agents) are the reason the `auth.log` / clear-text sniffing tricks above pay off. Combine credential capture here with the offline cracking workflow in [[Linux Dumping and Cracking Credentials]].

---

### Related notes
- [[Kerberos on Linux]] — request/steal/reuse tickets and keytabs from a domain-joined Linux host
- [[Clock Skew]] — sync time to the DC before any Kerberos-based attack
- [[AD Enumeration - Credentialed - From Linux]] — enumerate the domain from the attack box once you have creds
- [[AD Password Spraying]] — reuse harvested domain usernames
- [[Linux Dumping and Cracking Credentials]] — crack captured hashes/credentials offline