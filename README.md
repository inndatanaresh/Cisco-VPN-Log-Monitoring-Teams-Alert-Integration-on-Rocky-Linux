# Rocky Linux VPN Log Monitoring Setup

This document provides a step-by-step guide to set up VPN log monitoring on a Rocky Linux VM (IP: `10.10.5.36`), configure rsyslog to receive logs from a Cisco router, parse logs using a Python script, send alerts to Microsoft Teams, and automate log cleanup.

---

## 📌 Prerequisites

* Cisco Router configured to send syslog to `10.10.5.36` on UDP port 514
* Rocky Linux VM (`10.10.5.36`)
* Microsoft Teams Incoming Webhook URL

---

## 🛠️ Step 1: Install and Configure rsyslog

**Install and enable rsyslog:**

```bash
sudo dnf install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

**Configure rsyslog to receive logs:**
Create a new configuration file for Cisco logs:

```bash
sudo vi /etc/rsyslog.d/cisco.conf
```

Add the following content:

```rsyslog
# Listen to UDP syslog from Cisco
module(load="imudp")
input(type="imudp" port="514")

# Save Cisco logs to a separate file
if ($fromhost-ip == '106.51.5.232') then /var/log/cisco-vpn.log
& stop
```

**Restart rsyslog:**

```bash
sudo systemctl restart rsyslog
```

**Verify logs:**

```bash
tail -f /var/log/cisco-vpn.log
```

---

## 🛠️ Step 2: Create Python Script to Parse Logs and Send to Microsoft Teams

**Install Python and required packages:**

```bash
sudo dnf install python3-pip -y
pip3 install requests
sudo pip3 install requests
```

**Create a directory for the script:**

```bash
sudo mkdir -p /opt/vpn-alert
```

**Create the Python script:**

```bash
sudo nano /opt/vpn-alert/vpn_teams_alert.py
```

Add the following content:

````python
import time
import os
import requests

LOG_FILE = "/var/log/cisco-vpn.log"
WEBHOOK_URL = "<YOUR_WEBHOOK_URL_HERE>"

KEY_EVENTS = [
    "authenticated successfully",
    "session for", "is terminated",
    "fail", "failed", "invalid password", "authentication failed"
]

def send_to_teams(message):
    payload = {"text": message}
    try:
        r = requests.post(WEBHOOK_URL, json=payload)
        if r.status_code != 200:
            print(f"Error sending to Teams: {r.status_code} - {r.text}")
    except Exception as e:
        print(f"Exception while sending to Teams: {e}")

def monitor_log():
    print("Monitoring VPN login/logout/failure events...")
    with open(LOG_FILE, "r") as f:
        f.seek(0, os.SEEK_END)
        while True:
            line = f.readline()
            if not line:
                time.sleep(1)
                continue
            line_lower = line.lower()
            if any(event in line_lower for event in KEY_EVENTS):
                print(f"VPN event found: {line.strip()}")
                send_to_teams(f"🔐 **VPN Event**:\n```\n{line.strip()}\n```")

if __name__ == "__main__":
    monitor_log()
````

**Run the script:**

```bash
sudo python3 /opt/vpn-alert/vpn_teams_alert.py
```

---

## 🛠️ Step 3: Create a systemd Service for the Python Script

**Create a systemd service file:**

```bash
sudo nano /etc/systemd/system/vpn-teams-alert.service
```

Add the following content:

```ini
[Unit]
Description=VPN Log Monitoring for Microsoft Teams Alerts
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/vpn-alert/vpn_teams_alert.py
WorkingDirectory=/opt/vpn-alert
Restart=always
User=root
Group=root
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=vpn-teams-alert

[Install]
WantedBy=multi-user.target
```

**Reload systemd and enable the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable vpn-teams-alert
sudo systemctl start vpn-teams-alert
```

**Check the service status:**

```bash
sudo systemctl status vpn-teams-alert.service
```

---

## 🛠️ Step 4: Create a Cleanup Script for Old Logs

**Create the cleanup script:**

```bash
sudo nano /opt/vpn-alert/cleanup_vpn_logs.sh
```

Add the following content:

```bash
#!/bin/bash

LOG_FILE="/var/log/cisco-vpn.log"

# Delete logs older than 1 day
find $LOG_FILE -type f -mtime +1 -exec rm -f {} \;

echo "VPN logs older than one day have been deleted."
```

**Make the script executable:**

```bash
sudo chmod +x /opt/vpn-alert/cleanup_vpn_logs.sh
```

---

## 🛠️ Step 5: Set Up a Cron Job for Daily Cleanup

**Edit the crontab file:**

```bash
sudo crontab -e
```

**Add the following line to run the cleanup script daily at 2 AM:**

```cron
0 2 * * * /opt/vpn-alert/cleanup_vpn_logs.sh
```

**Verify the cron job:**

```bash
sudo crontab -l
```

You should see:

```cron
0 2 * * * /opt/vpn-alert/cleanup_vpn_logs.sh
```

---

## 📝 Notes

* Ensure the Cisco router IP (`106.51.5.232`) is correctly configured in the rsyslog configuration.
* Replace the `WEBHOOK_URL` in the Python script with your actual Microsoft Teams webhook URL.
* The cleanup script deletes logs older than one day. Adjust the `-mtime +1` parameter if a different retention period is needed.
* Monitor the service and logs regularly to ensure the system is functioning as expected.
