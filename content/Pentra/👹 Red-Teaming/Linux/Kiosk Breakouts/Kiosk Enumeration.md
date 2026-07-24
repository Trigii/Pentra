---
title: Kiosk Enumeration
draft: false
tags:
---
 
Interactive [_kiosks_](https://en.wikipedia.org/wiki/Interactive_kiosk) are computer systems that are generally intended to be used by the public for tasks such as Internet browsing, registration, or information retrieval.

They are installed on public places, designed with restrictive interfaces and offer limited functionality but they are connected to a backend system.

A [_thin client_](https://en.wikipedia.org/wiki/Thin_client) provides a limited interface to a powerful back-end system. This type of client may be physical, such as a self-contained [_Wyse Client_](https://www.dell.com/en-us/work/shop/wyse-endpoints-and-software/sc/cloud-client/thin-clients) or virtual, such as the [_Citrix_](https://www.citrix.com/products/citrix-virtual-apps-and-desktops/) _virtual desktop_.

We'll focus on [_Porteus Kiosk_](https://porteus-kiosk.org/) and well be using LOTL tools since we are on a restricted environment and we cant use our kali linux tools. Recommended:
- [_Living Off The Land Binaries and Scripts_](https://github.com/LOLBAS-Project/LOLBAS) (LOLBAS) project
- Unix/Linux-based [GTFOBins](https://gtfobins.github.io/) project

To interact with kiosks, we typically use mouse, screen, keyboard, etc. In this case we will interact with it using VNC.

Installation:
```bash
sudo apt install tigervnc-viewer
```

Run the client:
```bash
xtigervncviewer
```

Enter the IP address of the Kiosk VM to enter the initial Kiosk interface (limited functionality browser window):
![[Pasted image 20260714164610.png]]

# Kiosk Enumeration

Break Out the expected user experience:

1. Navigate all the pages presented by the kiosk

2. Use the right mouse button which would present various submenus or context menus we could explore. If its disabled for an application, try to open a different application and use the right click functionality.

3. Try combinations of left, right, and middle-clicking combined with Shift and Ctrl keys on various items in the interface such as links or menu options.

4. Try to escape the application by using [keyboard shortcuts](https://en.wikipedia.org/wiki/Table_of_keyboard_shortcuts) like Alt + Tab:
![[Pasted image 20260714165316.png]]

> [!Info]
> Keyboard shortcut lists are available online for a variety of operating systems including [Windows](https://support.microsoft.com/en-us/help/12445/windows-keyboard-shortcuts) and Linux window managers such as [Gnome](https://help.gnome.org/users/gnome-help/stable/keyboard-shortcuts-set.html.en) and [KDE](https://docs.kde.org/stable5/en/khelpcenter/fundamentals/kbd.html).

# Kiosk Browser Enumeration

- Clicking and holding down on the back button can show the browser history or any pages previously visited.

- We can also interact with the URL/address bar. By entering text into the URL bar, we are presented with suggested links for keywords that we type.

- We can interact with the preferences (gear) icon
![[Pasted image 20260714170338.png]]

- Many browsers include _keyword addresses_ which provide access to various functionality. We can use the [Firefox internal keywords](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/The_about_protocol), such as `about:config`

- We can see that the kiosks homepage URL begins with `file://`. This indicates that the content is stored locally on the kiosks filesystem. We can perform directory/file listing using this syntax:
![[Pasted image 20260714170815.png]]

> [!Note]
> We may be able to leverage directory listings or error messages caused by erroneous requests to gain information about the server process hosting the pages.
> 

- Explore other URIs like `chrome://`, `ftp://`, `mailto:`, 
`smb://`, `irc://`, etc. We can see that the `irc://` URI which uses the [irc protocol](https://www.w3.org/Addressing/draft-mirashi-url-irc-01.txt) to connect to text-based [Internet Relay Chat](https://en.wikipedia.org/wiki/Internet_Relay_Chat) (IRC) servers is very interesting:
![[Pasted image 20260714171436.png]]

---
# Command Execution

