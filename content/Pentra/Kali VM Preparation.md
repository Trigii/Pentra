---
title: Kali VM Preparation
draft: false
tags:
---
 
Software installation:
```bash
$ sudo apt update
$ sudo apt install gobuster
$ sudo apt install seclists
$ sudo apt install seclists curl dnsrecon enum4linux feroxbuster gobuster impacket-scripts nbtscan nikto nmap onesixtyone oscanner redis-tools smbclient smbmap snmp sslscan sipvicious tnscmd10g whatweb

sudo apt install libreoffice

AD exploitation tools: https://github.com/expl0itabl3/Toolies.git

https://github.com/tylerdotrar/SigmaPotato # use release and download .exe
https://github.com/X0RW3LL/XenSpawn # follow the steps 

Ruby:
sudo apt-get install build-essential patch
sudo apt-get install ruby-dev zlib1g-dev liblzma-dev libcurl4-openssl-dev

SNMP libraries:
sudo apt install snmp snmp-mibs-downloader rlwrap -y

SMTP Libre Office payload generator:
git clone https://github.com/0bfxgh0st/MMG-LO/

Rust:
sudo apt install cargo
cargo install feroxbuster

Git dumper:
pipx install git-dumper


Redis rogue server:
https://github.com/n0b0dyCN/redis-rogue-server.git

Spose (for pivoting though squid proxy)
https://github.com/aancw/spose.git

sudo apt install crackmapexec
sudo apt install ncat
sudo apt install chisel # windows version can be downloaded from github releases (chisel_windows_386 version)
sudo apt-get install keepassxc

sudo apt update && sudo apt install -y bloodhound
sudo bloodhound-setup

sudo apt install python3-wsgidav

Decrypt Autologon:
https://github.com/securesean/DecryptAutoLogon/blob/main/DecryptAutoLogon/bin/Release/DecryptAutoLogon.exe


sudo apt install mingw-w64 # for compiling cpp

+ PWSH +
sudo apt update
sudo apt install -y snapd # install daemon and tooling that enable snap packages
sudo systemctl enable --now snapd apparmor
*restart*
sudo snap install powershell --classic
----

sudo apt install burpsuite

Add Foxy proxy extension and configure the certificate (https://www.youtube.com/watch?v=lqfUclxl0K0)

Ligolo: 
https://github.com/nicocha30/ligolo-ng
sudo apt install ligolo-ng
which ligolo-agent
which ligolo-proxy
copy them to /opt/tools

LDAP:
sudo apt-get install libsasl2-dev python-dev-is-python3 libldap2-dev libssl-dev

git clone https://github.com/ropnop/windapsearch.git

Sigma potato:
wget https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe

Install:
- winpeas and linpeas
  
  
+ docker +
kali@kali:~$ sudo apt update
kali@kali:~$ sudo apt install -y docker.io
kali@kali:~$ sudo systemctl enable docker --now
kali@kali:~$ docker
sudo usermod -aG docker $USER

+ pspy +
Install pspy64: https://github.com/DominicBreuker/pspy/releases
Copy the binary to the target machine and execute it
```

Bloodhound installation error postgresql lib incompatibility:

#### 1. Refrescar la versión de collation (recomendado)

Ejecuta en `psql` como `postgres`:

`sudo -u postgres psql`

y dentro:

`ALTER DATABASE postgres REFRESH COLLATION VERSION;`

> [!Note]
> Cambia postgres por el nombre del resto de las bases de datos dentro listando con \l

Si BloodHound creó su propia base de datos, haz lo mismo para esa:

`\l   -- lista todas las bases de datos 
ALTER DATABASE bloodhound REFRESH COLLATION VERSION;`

`sudo vim /etc/bhapi/bhapi.json`

`admin:admin`

documentation that has to be improved:
1. AD (segmentate sections into different files)
2. 


Kali guest aditions:

https://www.youtube.com/watch?v=mWPPmaOXnrI

1. Delete virtualbox and the VM and everything
2. Follow VERY WELL the steps of the video until the installation of the linux headers (not included)
3. Run the following commands:
```
sudo apt-get update # This will update the repositories list
sudo apt-get upgrade # This will update all the necessary packages on your system
sudo apt-get dist-upgrade # This will add/remove any needed packages
reboot # You may need this since sometimes after a upgrade/dist-upgrade, there are some left over entries that get fixed after a reboot
sudo apt-get install linux-headers-$(uname -r) # This should work now
```

4. Follow again the steps of the video and it will be done :)

```
sudo apt-get install build-essential dkms
sudo apt-get install xserver-xorg xserver-xorg-core


$ apt-cache search linux-headers

```

![[Pasted image 20251012235405.png]]

# Smarll Window error

In case of small window:
1. Turn HDPI on
2. Restart VM with Command + R
3. Click on View
4. Put seamless mode on (window should disappear)
5. Restart VM again with Command + R (selecting the VM on the toolbar)
6. The window size should be OK. Turn off the hdpi mode and restart the machine
7. Turn again on the HDPI mode


# Upgrade packages and Getting guest additions not loading error

Error: the virtualbox kernel service is not running

Cause: upgrading packages (linux headers also upgrade and its incompatible with the current distribution and current guest additions)

Solution:
```
sudo apt-get update # This will update the repositories list
sudo apt-get upgrade # This will update all the necessary packages on your system
sudo apt-get dist-upgrade # This will add/remove any needed packages
reboot # You may need this since sometimes after a upgrade/dist-upgrade, there are some left over entries that get fixed after a reboot
sudo apt-get install linux-headers-$(uname -r) # This should work now
```

Follow the steps again of the video from the headers part:
https://www.youtube.com/watch?v=mWPPmaOXnrI

# Black window error

Prss Command + G or follow this tutorial:

https://www.youtube.com/watch?v=fDMHJn1c5Zc

