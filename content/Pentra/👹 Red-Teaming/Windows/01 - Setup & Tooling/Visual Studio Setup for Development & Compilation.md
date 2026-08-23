---
title: Visual Studio Setup for Development & Compilation
draft: false
tags:
  - red-teaming
  - windows
  - development
  - setup
  - c-sharp
---

This note documents the workflow for developing and compiling C# payloads (shellcode runners, injectors, DLL proxies) for Windows targets from a Kali attack machine, using a **Samba share** to bridge the filesystems. Visual Studio runs on a Windows VM with the project files stored on the shared Kali directory — edits are immediately visible to both sides.

> [!Note] When you need this
> Any technique requiring compiled C# or C++ output: [[Process Injection and Migration]], [[Phishing with Jscript (for emails)]] (DotNetToJScript), [[Client Side Attacks with File Containers (DLLs)]] (DLL proxy), or custom shellcode runners.

---
# SMB Share Setup

Backup configuration file and create a new one:
```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.old

sudo nano /etc/samba/smb.conf
```

Add the following content to smb.conf to create the visualstudio on our shared folder:
```
[visualstudio]
path = /media/sf_Shared
browseable = yes
read only = no
```

Create a Samba user and start the service:
```sh
$ sudo smbpasswd -a kali
$ sudo systemctl start smbd
$ sudo systemctl start nmbd
```

> [!Note]
> We must start the service every time we start our attack machine.

Create the shared folder and permissions:
```sh
$ mkdir /home/kali/data
$ chmod -R 777 /home/kali/data
```

Connect to our share:
```powershell
# RDP
Go to the File Explorer -> Network -> and type in the bar:
\\LOCAL_IP\visualstudio

# CLI
C:\> net use E: \\LOCAL_IP\visualstudio /u:kali kali
```

![[Pasted image 20260701181254.png]]

---
# Visual Studio Configuration

1. Open Visual Studio and create a new project:
![[Pasted image 20260701183740.png]]

> [!Note]
> For JScript, we select `Console App (.NET Framework)` project.
> For DLLs, we select `Class Library (.NET Framework)` project.

2. Set the location of the project (our SMB share folder):
![[Pasted image 20260701184158.png]]

3. Accept the default values and create the project.

Solution: parent unit that contain multiple projects.
Project: contains the app we are creating

**Compilation**

Switch the compilation from Debug to Release to avoid executing security scanners and display debug information:
![[Pasted image 20260701221235.png]]

We can now compile our application by navigating to _Build_ > _Build Solution_ or _Build_ > _Build ConsoleApp1_. We can view the output below:
![[Pasted image 20260701221350.png]]

> [!Important] Architecture targeting
> Always verify the architecture before compiling — most Windows targets are **x64**. Set it via:
> - **Configuration Manager** → Platform → `x64` (not `Any CPU` or `x86`)
> - Or: Project Properties → Build → Platform target → `x64`
>
> A mismatched architecture (e.g., x86 shellcode injected into an x64 process) causes the session to die silently.

Compiled output will be at:
```
\\ATTACKER_IP\visualstudio\ProjectName\bin\x64\Release\ProjectName.exe
```

**Project type reference:**

| Use case | Project template |
|----------|------------------|
| Shellcode runner / injector | Console App (.NET Framework) |
| DLL payload / proxy | Class Library (.NET Framework) |
| JScript via DotNetToJScript | Console App (.NET Framework) |

---
# Accessing Compiled Files from Kali

Project files live in `/media/sf_Shared` (mounted as `\\ATTACKER_IP\visualstudio`), so compiled binaries are immediately accessible from Kali without copying:

```bash
# List compiled output
$ ls /media/sf_Shared/ProjectName/bin/x64/Release/

# Serve to victim via HTTP
$ python3 -m http.server 80
```

---
# Related Notes
- [[Process Injection and Migration]] — C# process injection tools compiled here
- [[Phishing with Jscript (for emails)]] — DotNetToJScript and C# shellcode runners compiled here
- [[Client Side Attacks with File Containers (DLLs)]] — DLL proxy compilation
- [[Reflective PowerShell]] — in-memory alternative that skips compilation entirely
