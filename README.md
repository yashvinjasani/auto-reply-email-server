# auto-reply-email-server
Linux server running 24 hrs
# Auto-Reply Email Bot (IMAP/SMTP)

A lightweight, persistent email auto-responder written in Python. It monitors an inbox using IMAP, replies to new messages via SMTP, and uses a local SQLite database to prevent reply loops and ensure contacts are only emailed once per defined period (e.g., every 24 hours).

## Features

- **Smart Loop Prevention**: Detects `Auto-Submitted`, `Precedence: bulk`, and `noreply` headers to avoid replying to bots or mailing lists.
- **Memory (Stateful)**: Uses SQLite to track who has been emailed.
- **Cooldown System**: Configurable cooldown (default: 24 hours) prevents spamming the same sender multiple times.
- **Thread Aware**: Sets `In-Reply-To` headers so the auto-reply appears correctly in the sender's conversation thread.
- **Secure**: Designed to run with App Passwords and standard SSL/TLS.

## Prerequisites

- Python 3.6+
- An email account with IMAP/SMTP access (Gmail, Outlook, Postfix, etc.)
- *Note for Gmail users*: You must enable 2-Factor Authentication and generate an **App Password**.

## Installation

1. Clone the repository:
   ```bash
   git clone [https://github.com/yourusername/auto-reply-bot.git](https://github.com/yourusername/auto-reply-bot.git)
   cd auto-reply-bot
Configuration
Open bot.py and edit the Configuration section at the top:
                                                         EMAIL_USER = "your_email@example.com"
                                                      EMAIL_PASS = "your_app_password"
                                                      IMAP_SERVER = "imap.gmail.com"
                                                      SMTP_SERVER = "smtp.gmail.com"

Usage
Run the bot manually for testing:
                                  python3 bot.py

Deployment
For 24/7 background execution on a Linux server, see SERVER_INSTRUCTIONS.md.
---

### 2. `bot.py`
*Save this as `bot.py`. This is the core logic.*

```python
#!/usr/bin/env python3
import imaplib
import smtplib
import email
import time
import sqlite3
import os
from datetime import datetime, timedelta
from email.mime.text import MIMEText
from email.utils import parseaddr

# ==========================================
# CONFIGURATION
# ==========================================

# Email Server Settings (Example: Gmail)
EMAIL_USER = "your_email@example.com"
EMAIL_PASS = "your_password_or_app_password"
IMAP_SERVER = "imap.gmail.com"
SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 587

# Reply Settings
REPLY_COOLDOWN_HOURS = 24  # How long to wait before replying to the same person again
DB_FILE = "autoreply_log.db"

# The Content of your Auto-Reply
REPLY_SUBJECT = "Auto-Reply: I am currently out of office"
REPLY_BODY = """
Hello,

Thank you for your email. I am currently away from my inbox and will not be checking email regularly.
I will respond to your message as soon as possible upon my return.

Best regards,
Automated Server
"""

# ==========================================
# DATABASE LOGIC
# ==========================================

def init_db():
    """Initializes the SQLite database to track replies."""
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS reply_log
                 (email_address TEXT PRIMARY KEY, last_replied_at TIMESTAMP)''')
    conn.commit()
    conn.close()

def should_reply(email_address):
    """
    Returns True if we should reply (user not in DB or cooldown expired).
    Returns False if we replied recently.
    """
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT last_replied_at FROM reply_log WHERE email_address = ?", (email_address,))
    result = c.fetchone()
    conn.close()

    if result is None:
        return True # New contact

    last_reply_time = datetime.strptime(result[0], '%Y-%m-%d %H:%M:%S.%f')
    if datetime.now() - last_reply_time > timedelta(hours=REPLY_COOLDOWN_HOURS):
        return True # Cooldown expired
    
    return False

def log_reply(email_address):
    """Updates the database with the current timestamp for this contact."""
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO reply_log (email_address, last_replied_at) VALUES (?, ?)", 
              (email_address, datetime.now()))
    conn.commit()
    conn.close()

# ==========================================
# EMAIL LOGIC
# ==========================================

def send_auto_reply(to_address, original_msg_id):
    """Sends the actual email via SMTP."""
    msg = MIMEText(REPLY_BODY)
    msg['Subject'] = REPLY_SUBJECT
    msg['From'] = EMAIL_USER
    msg['To'] = to_address
    
    # Anti-Loop Headers (Standard)
    msg['Auto-Submitted'] = 'auto-replied'
    msg['Precedence'] = 'bulk'
    msg['X-Auto-Response-Suppress'] = 'All'
    
    if original_msg_id:
        msg['In-Reply-To'] = original_msg_id
        msg['References'] = original_msg_id

    try:
        server = smtplib.SMTP(SMTP_SERVER, SMTP_PORT)
        server.starttls()
        server.login(EMAIL_USER, EMAIL_PASS)
        server.sendmail(EMAIL_USER, to_address, msg.as_string())
        server.quit()
        
        log_reply(to_address)
        print(f"[{datetime.now()}] SENT reply to {to_address}")
        return True
    except Exception as e:
        print(f"[{datetime.now()}] ERROR sending to {to_address}: {e}")
        return False

def check_mail():
    """Connects to IMAP, searches for unseen mail, and processes it."""
    try:
        mail = imaplib.IMAP4_SSL(IMAP_SERVER)
        mail.login(EMAIL_USER, EMAIL_PASS)
        mail.select('inbox')

        status, messages = mail.search(None, 'UNSEEN')
        
        if status != 'OK' or not messages[0]:
            return # No new mail

        for e_id in messages[0].split():
            # Fetch headers only first to save bandwidth, or full RFC822 if needed
            res, msg_data = mail.fetch(e_id, '(RFC822)')
            
            for response_part in msg_data:
                if isinstance(response_part, tuple):
                    msg = email.message_from_bytes(response_part[1])
                    sender_name, sender_email = parseaddr(msg['From'])
                    message_id = msg.get('Message-ID')

                    # --- FILTERS ---
                    if sender_email == EMAIL_USER: continue
                    if 'noreply' in sender_email.lower(): continue
                    if msg.get('Auto-Submitted') == 'auto-generated': continue
                    if msg.get('Precedence') in ['bulk', 'list', 'junk']: continue

                    # --- DECISION ---
                    if should_reply(sender_email):
                        send_auto_reply(sender_email, message_id)
                    else:
                        print(f"[{datetime.now()}] SKIPPED {sender_email} (In cooldown)")

        mail.close()
        mail.logout()
    except Exception as e:
        print(f"[{datetime.now()}] CRITICAL ERROR: {e}")

if __name__ == "__main__":
    # Ensure DB exists
    init_db()
    print(f"--- Auto-Reply Bot Started [{datetime.now()}] ---")
    print(f"Monitoring {EMAIL_USER}...")
    
    while True:
        check_mail()
        # Wait 60 seconds before next check
        time.sleep(60)

3. SERVER_INSTRUCTIONS.md
Save this as SERVER_INSTRUCTIONS.md. This contains the Linux deployment steps.
# Linux Server Deployment Instructions

These instructions explain how to run the auto-reply bot as a background service (daemon) using `systemd`. This ensures the bot starts automatically when the server boots and restarts if it crashes.

## 1. Prepare the Directory

Login to your Linux server and create a folder for the application:

```bash
mkdir -p ~/email_bot
cp bot.py ~/email_bot/
cd ~/email_bot

2. Get the Absolute Paths
You need the absolute path to your Python executable and your script. Run this command inside your folder:
pwd
# Output example: /home/ubuntu/email_bot
If using a virtual environment, your python path is /home/ubuntu/email_bot/venv/bin/python. If using standard python, it is likely /usr/bin/python3.

3. Create the Service File
Create a systemd service configuration file:
sudo nano /etc/systemd/system/emailbot.service

Paste the following content (Edit User and Paths to match your system):
[Unit]
Description=Python Auto-Reply Email Bot
After=network.target

[Service]
# CHANGE THIS to your linux username
User=ubuntu
Group=ubuntu

# CHANGE THIS to your project folder path
WorkingDirectory=/home/ubuntu/email_bot

# CHANGE THIS to your python path and script path
ExecStart=/usr/bin/python3 /home/ubuntu/email_bot/bot.py

# Restart the script automatically if it crashes
Restart=always
RestartSec=10

# Ensure python logs are output to system journal immediately
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
Save and exit (Ctrl+O, Enter, Ctrl+X).

4. Start the Service
Reload the system daemon to recognize your new file:
sudo systemctl daemon-reload

Enable the service to start on boot:
sudo systemctl enable emailbot.service

Start the service immediately:
sudo systemctl start emailbot.service

Stopping/Updating
To stop the bot (e.g., to edit the code):
sudo systemctl stop emailbot.service

After editing bot.py, restart the service:
sudo systemctl restart emailbot.service
