---
title: Windows Kiosk Breakouts
draft: false
tags:
---
 
**Environment Variables**

If a browser-based kiosk accepts text input, we could substitute environment variables for full file paths.

For example, the _%APPDATA%_ variable translates to a local folder that stores data created by programs. If the kiosk has restricted filesystem browsing, we may be able to use this environment variable to browse the otherwise-protected locations on the filesystem:
![[Pasted image 20260715233524.png]]

Environment variables:

|Enviroment variable|Location|
|---|---|
|%ALLUSERSPROFILE%|C:\Documents and Settings\All Users|
|%APPDATA%|C:\Documents and Settings\Username\Application Data|
|%COMMONPROGRAMFILES%|C:\Program Files\Common Files|
|%COMMONPROGRAMFILES(x86)%|C:\Program Files (x86)\Common Files|
|%COMSPEC%|C:\Windows\System32\cmd.exe|
|%HOMEDRIVE%|C:\|
|%HOMEPATH%|C:\Documents and Settings\Username|
|%PROGRAMFILES%|C:\Program Files|
|%PROGRAMFILES(X86)%|C:\Program Files (x86) (only in 64-bit version)|
|%SystemDrive%|C:\|
|%SystemRoot%|C:\Windows|
|%TEMP% and %TMP%|C:\Documents and Settings\Username\Local Settings\Temp|
|%USERPROFILE%|C:\Documents and Settings\Username|
|%WINDIR%|C:\Windows|

**UNC Paths**

We can also enter full UNC paths in user input boxes or file browsers. We may be restricted from accessing **C:\Windows\System32**, but **\\127.0.0.1\C$\Windows\System32\** may be allowed.
![[Pasted image 20260715233626.png]]

**Shell Commands**

Windows also allows the use of the ["shell:"](https://ss64.com/nt/shell.html) shortcut in file browser dialogs to provide access to certain folders. Here are some [shell commands](https://www.winhelponline.com/blog/shell-commands-to-access-the-special-folders) available:

|Command|Action|
|---|---|
|**shell:System**|Opens the system folder|
|**shell:Common**|Start Menu Opens the Public Start Menu folder|
|**shell:Downloads**|Opens the current user's Downloads folder|
|**shell:MyComputerFolder**|Opens the "This PC" window, showing devices and drives for the system|

**Brower-Protocol style shortcuts**

We may also be able to use other [browser-protocol style shortcuts](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/jj710217\(v=vs.85\)) such as `file:///` to access applications or to access files that may open an application.

**Search Boxes**

Maybe entering a path to a specific file may be blocked, but it may be possible to search for files that we can't access directly:
![[Pasted image 20260715234621.png]]

Also, if we get access to a help dialog, we may be able to search for specific utilities such as Notepad, **cmd.exe**, or PowerShell:
![[Pasted image 20260715234901.png]]

If a clickable link is available in a search utility, we should try to take advantage of that by exploring the link and attempting various combinations of mouse-clicks and function-key clicks on the link.

**Shortcuts**

File shortcuts also offer interesting avenues for expansion as they may provide access to files and locations that are normally restricted.
For example, when using a file browser dialog in a kiosk, we may be able to create shortcuts by right-clicking on files and locations and choosing Create shortcut:
![[Pasted image 20260715235857.png]]

If this works, we may be able to modify the shortcut and change the target application in the shortcut properties to an application like **cmd.exe** or **powershell.exe** which could launch an interactive shell on the system

**Right-Clicking**

This approach also works with multiple special folders in file browser dialog windows. Right-clicking files in the file browser may present an option to add the file or a shortcut to "Favorites" or send it to a particular location. 

> [!Note]
> Because right-click functionality is widely used in Windows applications, it is difficult to restrict in a kiosk environment and should be attempted frequently as we increase our latitude on the system. These right-click menus are a common weakness in kiosk systems.

If we can browse the filesystem, such as through a file open or save dialog, but right-clicking is disabled, it may be possible to start an application by dragging and dropping files onto it (for example cmd.exe or powershell.exe). If the filetype being dragged is associated with the program, the program will likely open it.
![[Pasted image 20260716000648.png]]

**Printer**

The print dialog, if available in the kiosk, can provide a useful way of providing a working file browser dialog, even in extremely locked-down systems. Once a file browser is activated, we can use techniques similar to the previously ones to escape from the dialog and run applications or manipulate the filesystem:
![[Pasted image 20260716000904.png]]

**Shortcuts**

We can use various keyboard shortcuts to expand our level of access:
- `Ctrl + Alt + Del`: can launch the lock screen menu, which can allow us to log in as a different user or start Task Manager:
- `Ctrl + Alt + Esc`: can launch directly the Task Manager:

| Key | Menu/Application |
| --- | ---------------- |
| !   | Help             |
| C+P | Print Dialog     |
| E+A | Task Switcher    |
| G+R | Run menu         |
| C+~ | Start Menu       |

**Application Whitelisting/Blacklisting**

Windows systems may also include various application whitelisting or blacklisting strategies. There are many potential bypasses for these, for example, to copy and paste binaries, rename them, and attempt to run them:
![[Pasted image 20260716001344.png]]

Many blacklists/whitelists work on either a hash of the file, the filename, or the file path. Modifying any one of these will bypass blacklists. The reverse is true for whitelisting. If we have write access to a known whitelisted file, we can replace it with a binary that is normally restricted.