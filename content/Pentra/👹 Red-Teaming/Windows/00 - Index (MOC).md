---
title: Windows Red-Teaming — Index (MOC)
draft: false
tags:
  - red-teaming
  - windows
  - moc
  - osep
  - index
---

> [!info] Map of Content
> Central index for my Windows red-teaming / OSEP notes. Organized along the attack lifecycle: tooling → social engineering → delivery → AV evasion → AMSI/UAC → post-exploitation. Each entry links to the note and lists the primary MITRE ATT&CK technique(s) it covers.

---

# How this vault is organized

The folders are numbered to follow the operational flow of a client-side breach, which mirrors the OSEP progression:

1. **Setup & Tooling** — build/compile environment before anything else.
2. **Social Engineering & Phishing** — craft the pretext and the payload document.
3. **Delivery & MotW Bypass** — get the payload past zone/SmartScreen controls.
4. **Antivirus Evasion** — survive static (signature) and dynamic (heuristic) scanning.
5. **AMSI & UAC Bypass** — defeat in-memory script scanning and elevate.
6. **Post-Exploitation** — injection, migration, and in-memory execution once landed.

---

# 01 — Setup & Tooling

- [[Visual Studio Setup for Development & Compilation]] — dev/compile environment for the C# and .NET tooling used throughout.

# 02 — Social Engineering & Phishing

- [[Pretexting]] — lure design, authority/urgency/relevance, template repos.
- [[Phishing with Microsoft Office]] — VBA macro delivery + PowerShell shellcode runner.
- [[Phishing with Jscript]] — WSH/JScript execution, DotNetToJScript, in-memory .NET.
- [[Phishing with Calendars]] — `.ics` calendar-invite delivery vector.
- [[Client Side Attacks with File Containers (DLLs)]] — DLL side-loading via signed binaries shipped in ISO/ZIP containers.

# 03 — Delivery & MotW Bypass

- [[Mark of the Web (MotW)]] — how the zone identifier triggers Protected View / SmartScreen and how containers strip it.

# 04 — Antivirus Evasion

- [[Antivirus Evasion]] — overview / mental model for the whole AV-evasion chapter.
- [[Signature Based Detection]] — static signatures, string/byte matching, obfuscation & packing.
- [[Behavior-Heuristic Based Detection]] — dynamic/heuristic analysis, sandboxing, emulation.
- [[Bypassing AV in Office]] — macro obfuscation to survive document scanning.

# 05 — AMSI & UAC Bypass

- [[Antimalware Scan Interface (AMSI)]] — how AMSI hooks script engines; where it sits in the pipeline.
- [[Bypassing AMSI With Reflection in PowerShell]] — reflection-based patch of the AMSI context.
- [[Wrecking AMSI in PowerShell]] — alternative in-memory AMSI neutralization.
- [[Bypassing AMSI in JScript]] — AMSI bypass from the WSH/JScript engine.
- [[FodHelper UAC Bypass]] — registry-hijack elevation via `fodhelper.exe`.

# 06 — Post-Exploitation

- [[Application Whitelisting]] — AppLocker architecture, trusted-folder and DLL bypass, ADS execution.
- [[Bypassing AppLocker with PowerShell]] — CLM bypass via custom runspaces and InstallUtil.
- [[Bypassing AppLocker with C#]] — Microsoft.Workflow.Compiler and MSBuild arbitrary code execution.
- [[Bypassing AppLocker with JScript]] — MSHTA and XSL-Transform (Squiblytwo) execution vectors.
- [[Process Injection and Migration]] — classic/remote injection, migration, reflective PE loading (C#).
- [[Reflective PowerShell]] — in-memory shellcode execution without touching disk.

---

# MITRE ATT&CK mapping

Quick cross-reference to align each note with the framework — useful for report-writing and for structuring the OSEP exam methodology.

| Note | Tactic | Primary ATT&CK technique(s) |
|------|--------|-----------------------------|
| Pretexting | Initial Access | T1566 Phishing |
| Phishing with Microsoft Office | Initial Access / Execution | T1566.001, T1204.002, T1059.005 (VBA) |
| Phishing with Jscript | Initial Access / Execution | T1566.001, T1204.002, T1059.007 (JScript) |
| Phishing with Calendars | Initial Access | T1566.001 (invite delivery) |
| Client Side Attacks (DLLs) | Execution / Defense Evasion | T1574.002 DLL Side-Loading, T1553.005 |
| Mark of the Web (MotW) | Defense Evasion | T1553.005 Subvert Trust Controls: MotW Bypass |
| Antivirus Evasion (overview) | Defense Evasion | TA0005 (chapter overview) |
| Signature Based Detection | Defense Evasion | T1027, T1027.002 Software Packing |
| Behavior-Heuristic Based Detection | Defense Evasion | T1027, T1497 Sandbox Evasion |
| Bypassing AV in Office | Defense Evasion | T1027, T1204.002, T1059.005 |
| Antimalware Scan Interface (AMSI) | Defense Evasion | T1562.001 Impair Defenses |
| Bypassing AMSI w/ Reflection (PS) | Defense Evasion / Execution | T1562.001, T1059.001 |
| Wrecking AMSI in PowerShell | Defense Evasion / Execution | T1562.001, T1059.001 |
| Bypassing AMSI in JScript | Defense Evasion / Execution | T1562.001, T1059.007 |
| FodHelper UAC Bypass | Privilege Escalation | T1548.002 Bypass UAC |
| Process Injection and Migration | Defense Evasion / Priv-Esc | T1055 Process Injection (.001/.002/.012) |
| Reflective PowerShell | Defense Evasion / Execution | T1620 Reflective Code Loading, T1059.001 |
| Visual Studio Setup | — | Tooling (no technique) |

---

# OSEP study track

A recommended review order that builds concepts before combining them. Read top-to-bottom; each stage assumes the previous one.

**Stage 1 — Foundations.** [[Visual Studio Setup for Development & Compilation]] → [[Antivirus Evasion]] → [[Signature Based Detection]] → [[Behavior-Heuristic Based Detection]]. Get the environment and the detection model straight first.

**Stage 2 — Client-side delivery.** [[Pretexting]] → [[Phishing with Microsoft Office]] → [[Phishing with Jscript]] → [[Phishing with Calendars]] → [[Client Side Attacks with File Containers (DLLs)]]. The core initial-access chapter.

**Stage 3 — Getting past the gate.** [[Mark of the Web (MotW)]] → [[Bypassing AV in Office]]. Tie delivery to evasion.

**Stage 4 — In-memory & AMSI.** [[Antimalware Scan Interface (AMSI)]] → [[Bypassing AMSI With Reflection in PowerShell]] → [[Wrecking AMSI in PowerShell]] → [[Bypassing AMSI in JScript]]. The most heavily-tested area.

**Stage 5 — Elevation & post-ex.** [[FodHelper UAC Bypass]] → [[Reflective PowerShell]] → [[Process Injection and Migration]].

> [!tip] Exam methodology
> For each technique, keep three things in your notes: (1) *why the detection fires*, (2) *the minimal change that defeats it*, and (3) *the defensive signal it leaves behind*. The blue-team angle is what makes the technique stick and is directly useful in the OSEP report.

---

# Maintenance notes — broken / missing wikilinks

Found while indexing. These render as dead links in Obsidian/Quartz:

- `[[AV Evasion]]` — referenced in several notes, but the actual file is **[[Antivirus Evasion]]**. Simple rename mismatch; safe to fix across the vault.
- `[[Client-Side Attacks]]` — referenced as an overview note that does not exist here. Either create a stub overview or repoint to [[Client Side Attacks with File Containers (DLLs)]].
- `[[Windows File Transfer]]` — referenced from [[Process Injection and Migration]]; note lives elsewhere in the vault (not in this folder).
- `[[Command and Control (C2-C&C)]]` — referenced from post-ex notes; also outside this folder.

Heading-anchor links (e.g. `[[Phishing with Microsoft Office#Mark of the Web]]`) resolve correctly.

**Idea: Framework de Automatización con Templates (diseño conceptual)**

> [!Important] Nota de Diseño
> Un framework de templates para automatizar la generación de notas de red-teaming. No es una herramienta ofensiva sino un sistema de documentación estructurado.
>
> **Propuesta de estructura:**
> ```
> templates/
> ├── shellcode_runner.cs.tmpl      # plantilla C# con placeholders {{LHOST}}, {{LPORT}}, {{KEY}}
> ├── vba_macro.vba.tmpl            # plantilla VBA con {{ENCRYPTION_KEY}}, {{ATTACKER_IP}}
> ├── powershell_runner.ps1.tmpl    # plantilla PS con {{AMSI_BYPASS}}, {{CRADLE_URL}}
> └── metadata.yaml                 # LHOST, LPORT, KEY centralizados
> ```
>
> **Herramientas que ya hacen esto (referencias):**
> - [Villain](https://github.com/t3l3machus/Villain) — framework de C2 con generación de payloads parametrizados
> - [SharPyShell](https://github.com/antonioCoco/SharPyShell) — templates C# para webshells
> - [OSEP-Code-Snippets](https://github.com/chvancooten/OSEP-Code-Snippets) — colección de snippets del curso OSEP con scripts de generación
>
> En lugar de construir el framework desde cero, el repositorio **OSEP-Code-Snippets** de chvancooten es exactamente lo que describes: todos los snippets del curso parametrizados y listos para usar. Estudíalo como referencia para el diseño de tu propio sistema de notas.