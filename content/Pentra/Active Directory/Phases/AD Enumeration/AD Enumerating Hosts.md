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