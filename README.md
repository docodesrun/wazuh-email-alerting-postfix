# wazuh-email-alerting-postfix
# Wazuh Email Alerting via Gmail SMTP Relay (Postfix)

A complete guide to setting up email alerts in Wazuh SIEM using Gmail as the SMTP relay through Postfix — including real errors encountered, how they were debugged, and a live alert analysis.

Built and tested on a VMware home lab running Wazuh Manager on Ubuntu 24.04 with a Windows 11 agent.

---

## 📋 Table of Contents

- [Introduction](#introduction)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1 — Generate Gmail App Password](#step-1--generate-gmail-app-password)
- [Step 2 — Install and Configure Postfix](#step-2--install-and-configure-postfix)
- [Step 3 — Configure Wazuh ossec.conf](#step-3--configure-wazuh-ossecconf)
- [Step 4 — Restart Services and Test](#step-4--restart-services-and-test)
- [Troubleshooting](#troubleshooting)
- [Real Alert Analysis](#real-alert-analysis)
- [Best Practices](#best-practices)
- [Lab Environment](#lab-environment)

---

## Introduction

Wazuh is a free, open-source security platform that monitors your endpoints and generates alerts based on security rules. But alerts sitting in a dashboard are only useful when someone is actively watching.

Email alerting pushes critical alerts directly to your inbox the moment they fire — whether you're at your desk or not.

This guide covers the full setup, the exact errors you'll likely hit, and how to fix them.

---

## Architecture

Wazuh Agent (Windows/Linux)
↓
Wazuh Manager (Ubuntu)
↓
wazuh-maild
↓
Postfix(localhost) <- handles TLS + Auth 
↓
Gmail SMTP(smtp.gmail.com:587)
↓
Your Inbox

**Why Postfix?**
Wazuh's built-in mail daemon (`wazuh-maild`) has no native support for TLS or SMTP authentication. Gmail requires both. Postfix sits in the middle and handles the authenticated relay to Gmail on Wazuh's behalf.

---

## Prerequisites

- Wazuh Manager installed and running
- Ubuntu 22.04 / 24.04
- At least one Wazuh agent connected
- A Gmail account with **2-Factor Authentication enabled**
- Gmail App Password generated (not your regular Gmail password)

---

## Step 1 — Generate Gmail App Password

1. Go to your Google Account → **Security**
2. Under "How you sign in to Google", click **2-Step Verification**
3. Scroll down and click **App Passwords**
4. Select app: **Mail** → Select device: **Linux Machine**
5. Click **Generate** and copy the 16-character password

> ⚠️ Save this password immediately — Google won't show it again.

---

## Step 2 — Install and Configure Postfix

### Install Postfix and mailutils

```bash
sudo apt update
sudo apt install postfix mailutils libsasl2-modules -y
```

During installation, select **Internet Site** and enter your hostname.

### Configure main.cf

```bash
sudo nano /etc/postfix/main.cf
```

Add or update the following lines:

```ini
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
smtp_tls_CApath = /etc/ssl/certs
```

### Create SASL credentials file

```bash
sudo nano /etc/postfix/sasl_passwd
```

Add this line (replace with your Gmail and App Password):
[smtp.gmail.com]:587    yourGmail@gmail.com:your_app_password

### Secure and compile the credentials

```bash
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
```

### Restart Postfix

```bash
sudo systemctl restart postfix
```

### Test Postfix relay

```bash
echo "Postfix relay test" | mail -s "Test Mail" your@email.com
```

Check the mail log:

```bash
tail -f /var/log/mail.log
```

Look for `status=sent (250 2.0.0 OK)` to confirm it's working.

---

## Step 3 — Configure Wazuh ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Update the `<global>` block:

```xml
<global>
  <email_notification>yes</email_notification>
  <smtp_server>localhost</smtp_server>
  <email_from>yourGmail@gmail.com</email_from>
  <email_to>your-inbox@yourdomain.com</email_to>
  <email_maxperhour>12</email_maxperhour>
  <email_alert_level>7</email_alert_level>
</global>
```

> ⚠️ `smtp_server` must be `localhost` — not `smtp.gmail.com`. Wazuh hands off to Postfix, which handles Gmail.

> ⚠️ `email_from` must match the Gmail account in your `sasl_passwd` file.

---

## Step 4 — Restart Services and Test

```bash
sudo systemctl restart postfix
sudo systemctl restart wazuh-manager
```

Monitor for email activity:

```bash
sudo tail -f /var/ossec/logs/ossec.log | grep -i mail
```

---

## Troubleshooting

### Error 1 — Mail from not accepted by server

wazuh-maild: ERROR: (1764): Mail from not accepted by server

**Cause:** Wazuh was pointing directly to `smtp.gmail.com`. Gmail rejected the connection because no authentication was provided.

**Fix:** Set `<smtp_server>localhost</smtp_server>` in `ossec.conf` and let Postfix handle the relay.

---

### Error 2 — Error Sending email to SMTP server

wazuh-maild: ERROR: (1263): Error Sending email to 192.178.211.109 (smtp server)

**Cause:** Same root cause — direct connection to Gmail's IP was being rejected at the SMTP level.

**Fix:** Same as above. Route through Postfix on localhost.

---

### Error 3 — No emails despite test mail working

**Cause:** Test mail sent via Postfix directly works fine. But if `ossec.conf` still points to `smtp.gmail.com`, Wazuh's maild bypasses Postfix entirely and fails silently.

**Fix:** Always confirm `smtp_server` is set to `localhost` in `ossec.conf`.

---

### How to read the logs for diagnosis

```bash
# Wazuh mail errors
sudo grep -i "email\|mail" /var/ossec/logs/ossec.log | tail -30

# Postfix relay status
sudo tail -f /var/log/mail.log
```

---

## Real Alert Analysis

During testing, the following real alert was captured from a Windows 11 agent:

Rule: 60602 (level 9) -> 'Windows application error event.'
Rule: 61061 (level 10) -> 'Multiple Windows error application events.'
Provider: ESENT
Event ID: 482
Computer: Tushar-Patil
Error: There is not enough space on the disk.
File: C:\WINDOWS\system32\CatRoot2\edbres00002.jrs

**What this means:**
ESENT (Extensible Storage Engine) is Windows' built-in database engine used by Windows Update, Windows Defender, and other system services. When the disk runs out of space, ESENT can't write to its journal files, causing repeated errors.

**Why it matters:**
Disk exhaustion can be a symptom of ransomware activity (T1485 — Data Destruction in MITRE ATT&CK), runaway logging, or simply neglected disk hygiene. Wazuh caught and escalated this to level 10 automatically.

**Recommended response:**
- Check disk usage on the affected machine
- Identify what's consuming space
- Clear temporary files and old logs
- Set up a low disk space monitoring rule in Wazuh

---

## Best Practices

- Always use a **Gmail App Password** — never your real Gmail password
- Secure your credentials file:
```bash
  sudo chmod 600 /etc/postfix/sasl_passwd
```
- Tune `email_alert_level` based on your environment — level 7 is a good starting point
- Tune `email_maxperhour` to avoid inbox flooding during noisy alert periods
- Add suppression rules in `local_rules.xml` for repetitive low-value alerts
- Regularly check `/var/ossec/logs/ossec.log` for mail errors

---

## Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation |
| Wazuh Manager OS | Ubuntu 24.04 LTS |
| Wazuh Version | 4.x |
| Agent OS | Windows 11 |
| Mail Relay | Postfix → Gmail SMTP |
| SMTP Port | 587 (STARTTLS) |

---





- Wazuh dashboard showing fired alerts
- Email received in inbox with alert details
- `ossec.log` showing successful mail delivery
- `mail.log` showing `status=sent`

---

## References

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Postfix Gmail Relay Guide](http://www.postfix.org/SASL_README.html)
- [MITRE ATT&CK T1485 — Data Destruction](https://attack.mitre.org/techniques/T1485/)

---

## Author

**Aryan Kandwal**
IT & Cybersecurity | Wazuh SIEM | Home Lab Builder
[LinkedIn](https://www.linkedin.com/in/aryankandwal?utm_source=share_via&utm_content=profile&utm_medium=member_android) | 
