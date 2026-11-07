---
title: Antivirus Evasion
draft: false
tags:
  - red-teaming
  - offensive
  - windows
  - av-evasion
  - evasion
---
 
Antivirus Evasion is the set of techniques used to prevent a malicious payload (stager, dropper, implant) from being detected and quarantined by endpoint security products. Modern AV/EDR combine several detection engines, so a robust evasion strategy usually has to defeat more than one of them.

# Detection Methods

AV products generally rely on three complementary detection strategies. Each one is bypassed differently:

- **Signature based detection**: matches static patterns (file hashes, byte sequences, YARA-style rules) against a database of known-bad samples. See [[Signature Based Detection]].
- **Behavior / heuristic based detection**: emulates or monitors the sample at runtime and flags suspicious actions (process injection, unusual API calls). See [[Behavior-Heuristic Based Detection]].
- **Heuristic in sandboxes / emulation**: runs the sample in a simulated environment before allowing execution. Bypassed with sandbox-detection tricks (sleep timers, environment checks) — also covered in [[Behavior-Heuristic Based Detection]].

# Testing Payloads

Check our payload against AV databases **before** using it on an engagement:
- [_VirusTotal_](https://www.virustotal.com/gui/home/upload): complete signature based DBs but it distributes findings to AV vendors (burns the sample).
- [_AntiScan.Me_](https://antiscan.me/): contains multiple DBs and does **not** distribute its findings, so it is safer for testing operational payloads.

> [!WARNING]
> Never upload a payload you intend to use in a live engagement to VirusTotal. Once submitted, the hash and bytes are shared with vendors and your implant will be signatured within hours.

# Common Evasion Approaches

- **Encoders / crypters**: obfuscate the payload bytes (e.g. `msfvenom -e x86/shikata_ga_nai -i 10`). Encoding alone rarely defeats modern AV but helps against naive signatures.
- **Custom shellcode runners**: write your own loader in C#/C++ that allocates memory and executes shellcode, avoiding known-bad templates. See [[Process Injection and Migration]] and [[Reflective PowerShell]].
- **Living off the land**: use trusted signed binaries (LOLBins) instead of dropping an executable.
- **Payload staging in memory**: avoid touching disk to defeat static scanning entirely.

# Related Notes

- [[Signature Based Detection]]
- [[Behavior-Heuristic Based Detection]]
- [[Bypassing AV in Office]]
- [[Client Side Attacks with File Containers (DLLs)]]

