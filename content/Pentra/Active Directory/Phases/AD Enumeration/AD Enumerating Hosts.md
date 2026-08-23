---
title: AD Enumerating Hosts
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
# Identifying Hosts 

- Wireshark: look for ARP requests and replies, Multicast DNS (MDNS) and other L2 protocols
```bash
$ sudo -E wireshark
```

> [!Note]
> If we are on a host without a GUI (which is typical), we can use [tcpdump](https://linux.die.net/man/8/tcpdump), [net-creds](https://github.com/DanMcInerney/net-creds), and [NetMiner](https://www.netminer.com/en/product/netminer.php), etc., to perform the same functions.

We can also use tcpdump to save a capture to a .pcap file, transfer it to another host, and open it in Wireshark.

```shell
$ sudo tcpdump -i INTERFACE 
```

- [Responder](https://github.com/lgandx/Responder-Windows) is a tool built to listen, analyze, and poison `LLMNR`, `NBT-NS`, and `MDNS` requests and responses. This means will listen for any resolution requests, but will not answer them or send out poisoned packets.:
```bash
$ sudo responder -I INTERFACE -A 

Parameters:
-A: analyze mode
```

> [!Note]
> Check [[AD Initial Foothold]] for how to use responder on an active mode and poison any responses.

- [Fping](https://fping.org/) provides us with a similar capability as the standard ping application in that it utilizes ICMP requests and replies to reach out and interact with a host. Where fping shines is in its ability to issue ICMP packets against a list of multiple hosts at once and its scriptability. Also, it works in a round-robin fashion, querying hosts in a cyclical manner instead of waiting for multiple requests to a single host to return before moving on.
```shell
$ fping -asgq SUBNET

Parameters:
`a`: to show targets that are alive
`s`: to print stats at the end of the scan
`g`: to generate a target list from the CIDR network, 
`q`: to not show per-target results.
```

> [!Note]
> Collect all information and store it into a single IP address file for the nmap scan

- Nmap scan:
```bash
sudo nmap -v -A -iL hosts.txt -oN /home/htb-student/Documents/host-enum
```

> [!Tip]
> Once a candidate list of live hosts exists, feed it into a focused sweep for the ports that reveal an AD environment (DNS, Kerberos, LDAP, SMB, WinRM):
> ```bash
> $ sudo nmap -Pn -n -p 53,88,135,139,389,445,464,636,3268,3269,5985 -iL hosts.txt -oA ad-hosts
> ```
> Host `88/tcp` (Kerberos) open + `389/tcp` (LDAP) usually flags a **Domain Controller**.

Quickly spot Domain Controllers with a broadcast/ICMP-free approach using nslookup or the SRV records once a DNS server is known:
```bash
# Locate DCs via DNS SRV records (needs the domain name + a DC acting as DNS)
$ nslookup -type=SRV _ldap._tcp.dc._msdcs.<DOMAIN>
$ dig @<DC_IP> _ldap._tcp.dc._msdcs.<DOMAIN> SRV +short
```

From Windows (living-off-the-land, no tools dropped):
```powershell
PS C:\> nltest /dclist:<DOMAIN>
PS C:\> Get-ADDomainController -Filter * | Select Name,IPv4Address,Site
```

> [!Note]
> `net-creds` / `Responder -A` passively harvest hashes and hostnames while you enumerate — leave one running in the background during the whole engagement.

---

### Related notes
- [[Host Discovery]] — generic (non-AD) live-host discovery techniques.
- [[Port Scanning]] — turning the host list into a service map.
- [[NetBIOS]] — `<1C>`/`<1B>` suffixes reveal DCs and the master browser.
- [[SMB]] — enumeration of the file-server / DC once port 445 is confirmed.
- [[AD Initial Foothold]] — flipping Responder to active mode to poison LLMNR/NBT-NS and capture [[NetNTLMv2]] hashes.
- [[AD Automatic Enumeration (BloodHound)]] — automated collection once you can authenticate.