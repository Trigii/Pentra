---
title: AD External Recon and Enumeration Principles
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - passive
---
 
We are looking for:
- IP Space
- Domain Information
- Schema Format
- Data Disclosures
- Breach Data

---
# Finding Address Spaces

The `BGP-Toolkit` hosted by [Hurricane Electric](http://he.net/) is a fantastic resource for researching what address blocks are assigned to an organization and what ASN they reside within. Just punch in a domain or IP address, and the toolkit will search for any results it can. We can glean a lot from this info. Many large corporations will often self-host their infrastructure, and since they have such a large footprint, they will have their own ASN. This will typically not be the case for smaller organizations or fledgling companies. As you research, keep this in mind since smaller organizations will often host their websites and other infrastructure in someone else's space (Cloudflare, Google Cloud, AWS, or Azure, for example). Understanding where that infrastructure resides is extremely important for our testing. We have to ensure we are not interacting with infrastructure out of our scope.

---
# DNS
DNS is a great way to validate our scope and find out about reachable hosts the customer did not disclose in their scoping document. Sites like [domaintools](https://whois.domaintools.com/), and [viewdns.info](https://viewdns.info/) are great spots to start. We can get back many records and other data ranging from DNS resolution to testing for DNSSEC and if the site is accessible in more restricted countries.

Validate nameservers:
```bash
$ nslookup NAMESERVER_FQDN
```

---
# Public Data

Google dorks:
- `filetype:pdf inurl:DOMAIN` (documents)
- `intext:"@DOMAIN" inurl:DOMAIN` (email addresses)

Tools like [Trufflehog](https://github.com/trufflesecurity/truffleHog) and sites like [Greyhat Warfare](https://buckets.grayhatwarfare.com/) are fantastic resources for finding these breadcrumbs.

# Username Harvesting
We can use a tool such as [linkedin2username](https://github.com/initstring/linkedin2username) to scrape data from a company's LinkedIn page and create various mashups of usernames (flast, first.last, f.last, etc.) that can be added to our list of potential password spraying targets.

# Credential Hunting
[Dehashed](http://dehashed.com/) is an excellent tool for hunting for cleartext credentials and password hashes in breach data.

```shell
$ sudo python3 dehashed.py -q DOMAIN_FQDN -p
```

> [!Note]
> The script used in the example above can be found [here](https://github.com/mrb3n813/Pentest-stuff/blob/master/dehashed.py). Due to changes in the API structure of DeHashed, modifications may be necessary. Alternatively, the following [script](https://github.com/sm00v/Dehashed) could be used. Before executing the script, it is crucial to become familiar with its functionality.