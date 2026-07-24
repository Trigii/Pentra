---
title: Pretexting
draft: false
tags:
  - red-teaming
  - social-engineering
  - phishing
  - pretexting
---

Pretexting is the practice of constructing a fabricated scenario (the "pretext") to manipulate a target into taking a desired action — opening a document, clicking a link, enabling macros, or disclosing credentials. The scenario must appear plausible and relevant to the target's work context to succeed.

Good pretexts exploit urgency, authority, and relevance. A target is far more likely to act when a request appears to come from IT, HR, or senior management and requires immediate action.

**Resources:**
- Templates repo: https://github.com/martinsohn/Office-phish-templates
- More templates: https://github.com/L4bF0x/PhishingPretexts

---
# Common Pretexts

## IT Department / Security
High-authority sender, triggers urgency around compliance or security:
- "Your account has been flagged for suspicious activity — review the attached report"
- "Mandatory security policy update — please review and acknowledge"
- "Your VPN certificate is expiring — run the attached renewal tool"
- "Password policy change — your current password must be reset via the attached form"

## HR Department
Widely trusted, sent to all employees — high-value for broad targeting:
- "Updated employee benefits package — review the attached document"
- "Performance review schedule for Q3 — please confirm your slot"
- "Urgent: payroll update requires your attention before end of month"
- "New remote work policy — action required by Friday"

## Finance / Accounting
Effective for spear-phishing specific finance roles:
- "Invoice #4821 requires approval — attached for your review"
- "Q3 expense report — please verify and sign off"
- "Wire transfer confirmation needed — see attached authorization"

## Management / Executive (BEC-style)
Spoofed CEO/manager email for high-value targets:
- "Can you open the attached contract before the board meeting today?"
- "Urgent — need you to review this before I get on my flight"

---
# Convincing the Victim to Enable Macros

When a victim opens a macro-enabled Office document from the internet, Microsoft Word shows a yellow **Protected View** or **Security Warning** bar. The key social engineering challenge is convincing them to click **"Enable Content"** or **"Enable Editing"**.

## Common Lure Techniques

**Fake blur/locked content overlay:** The document body contains a blurred or greyed-out image with text like:
> "This document is protected. To view the contents, click **Enable Content** above."

This mimics legitimate DRM-protected documents and is highly effective.

**Fake DocuSign / Adobe template:** Style the document to look like a DocuSign or Adobe Sign document, with a message:
> "To complete your electronic signature, please enable macros."

**Fake software update prompt:** Create a document that looks like a Microsoft Office update notification:
> "Your version of Office is out of date. Enable content to update."

**QR code / image display trick:** Tell the recipient the document contains an image or QR code that requires macros to render:
> "Enable content to display the embedded QR code for meeting access."

## The "Enable Content" Flow

```
Victim opens .docm / .doc
       ↓
Office shows yellow bar: [!] SECURITY WARNING: Macros have been disabled
       ↓
Victim clicks [Enable Content]  ← social engineering objective
       ↓
Document_Open / AutoOpen triggers
       ↓
Payload executes
```

> [!Note]
> If the document is delivered via **SMB share** (`\\attacker\share\doc.docm`) rather than email/download, it may bypass Protected View entirely — no "Enable Content" prompt needed. See [[Phishing with Microsoft Office#Mark of the Web]] for details.

---
# Document Design Tips

A convincing phishing document should:
- **Match the organization's branding** — use their logo, colors, and fonts if obtained via OSINT
- **Use realistic metadata** — set author name to a real employee (extracted via `exiftool` from public documents, see [[Client-Side Attacks]])
- **Match the file name to the pretext** — `Q3_Payroll_Report.doc`, `VPN_Certificate_Renewal.docm`, `Performance_Review_2024.doc`
- **Keep content minimal** — less content means less scrutiny; a blurred overlay with an "Enable Content" prompt is enough
- **Use `.doc` (Word 97-2003) over `.docm`** — `.doc` is less visually alarming and some email filters allow it more readily

---
# OPSEC for Delivery

- Send from a **lookalike domain** (e.g., `corp-hr@company-hr.com` instead of `hr@company.com`) or a compromised internal SMTP server for maximum trust
- Use **swaks** to send emails directly via SMTP: `sudo swaks -t target@company.com --from hr@company.com --server SMTP_IP --attach @document.doc --body @body.txt --header "Subject: Q3 Benefits Update"`
- Set `Reply-To` to a controlled inbox so any replies are captured
- Check that SPF/DKIM pass for your sending domain or server to avoid spam filters
- Personalize subject lines and bodies with the target's name and department (spear phishing)

---
# Related Notes
- [[Phishing with Microsoft Office]] — VBA macros and Office payload delivery
- [[Phishing with Calendars]] — calendar invite phishing vector
- [[Client-Side Attacks]] — overview of all client-side attack vectors
- [[AV Evasion]] — macro obfuscation to survive AV scanning of the document
