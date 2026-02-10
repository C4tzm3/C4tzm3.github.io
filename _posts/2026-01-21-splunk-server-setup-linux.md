---
layout: post
title: "Splunk Server Setup Guide - Linux"
date: 2026-01-21
categories: notes
tags: [splunk, linux, security, siem]
---

# Splunk Server Setup Guide - Linux Environment

Complete guide for setting up Splunk Enterprise Server with deployment capabilities.

## Installation Script

### Configuration Variables

Edit these at the top before running:

```bash
DEPLOYMENT_SERVER_IP="167.172.73.162"
SPLUNK_ADMIN="admin"
SPLUNK_PASSWORD="admin123!"
TIMEZONE="Asia/Kuala_Lumpur"
```

### Full Installation Script

```bash
#!/bin/bash
set -e

# ============================================
# CONFIGURATION - EDIT BEFORE RUNNING
# ============================================
DEPLOYMENT_SERVER_IP="167.172.73.162"
SPLUNK_ADMIN="admin"
SPLUNK_PASSWORD="admin123!"
TIMEZONE="Asia/Kuala_Lumpur"
# ============================================

SPLUNK_VERSION="10.0.2"
SPLUNK_BUILD="e2d18b4767e9"
SPLUNK_DEB="splunk-${SPLUNK_VERSION}-${SPLUNK_BUILD}-linux-amd64.deb"
SPLUNK_URL="https://download.splunk.com/products/splunk/releases/${SPLUNK_VERSION}/linux/${SPLUNK_DEB}"
SPLUNK_USER="splunk"
SPLUNK_HOME="/opt/splunk"

echo "=== Splunk Server Installation ==="

echo "[1/10] Setting timezone..."
sudo timedatectl set-timezone $TIMEZONE

echo "[2/10] Updating system..."
sudo apt update && sudo apt upgrade -y

echo "[3/10] Installing tools..."
sudo apt install -y wget net-tools ufw vim

echo "[4/10] Downloading Splunk..."
cd /tmp
wget -O $SPLUNK_DEB "$SPLUNK_URL"

echo "[5/10] Installing Splunk..."
sudo dpkg -i $SPLUNK_DEB
rm $SPLUNK_DEB

echo "[6/10] Starting Splunk..."
sudo $SPLUNK_HOME/bin/splunk start --accept-license --answer-yes

echo "[7/10] Configuring firewall..."
sudo ufw allow 8000/tcp   # Web interface
sudo ufw allow 9997/tcp   # Receiving port
sudo ufw allow 8089/tcp   # Management port
sudo ufw --force enable
sudo ufw reload

echo "[8/10] Creating splunk user..."
sudo useradd -m -s /bin/bash $SPLUNK_USER 2>/dev/null || true

echo "[9/10] Setting ownership..."
sudo $SPLUNK_HOME/bin/splunk stop
sudo chown -R $SPLUNK_USER:$SPLUNK_USER $SPLUNK_HOME
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk enable boot-start -user $SPLUNK_USER --accept-license --answer-yes

echo "[10/10] Enabling deployment server..."
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk enable deploy-server -auth $SPLUNK_ADMIN:$SPLUNK_PASSWORD

sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk restart

echo ""
echo "=== Installation Complete! ==="
echo "Web Interface: http://$(hostname -I | awk '{print $1}'):8000"
echo "Username: $SPLUNK_ADMIN"
echo "Password: $SPLUNK_PASSWORD"
echo ""
echo "IMPORTANT: Complete the manual configuration steps below!"
```

## Post-Installation Manual Configuration

### Step 1: Enable Data Receiving (Port 9997)

#### Option A: Via Web Interface (Recommended)

1. Login to Splunk Web at `http://YOUR_SERVER_IP:8000`
2. Go to **Settings > Forwarding and receiving**
3. Click **Configure receiving**
4. Click **New Receiving Port**
5. Enter port: `9997`
6. Click **Save**

#### Option B: Via Command Line

```bash
sudo -u splunk /opt/splunk/bin/splunk enable listen 9997 -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk restart
```

### Step 2: Create Indexes

#### Via Web Interface

1. Go to **Settings > Indexes**
2. Click **New Index**
3. Create these indexes:
   - **Index Name:** `sysmon` | **Max Size:** 500GB
   - **Index Name:** `security` | **Max Size:** 500GB
   - **Index Name:** `windows` | **Max Size:** 500GB
   - **Index Name:** `linux` | **Max Size:** 500GB

#### Via Command Line

```bash
sudo -u splunk /opt/splunk/bin/splunk add index sysmon -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk add index security -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk add index windows -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk add index linux -auth admin:admin123!
```

### Step 3: Install Splunk Add-ons

Install these apps to properly parse Windows and Sysmon logs:

1. Go to **Apps > Find More Apps**
2. Search and install:
   - **Splunk Add-on for Microsoft Windows** - Parses Windows Event Logs
   - **Splunk Add-on for Sysmon** - Parses Sysmon logs with field extraction
   - **Splunk Add-on for Unix and Linux** - For Linux log parsing

After installation:

```bash
sudo -u splunk /opt/splunk/bin/splunk restart
```

### Step 4: Configure Deployment Server (Optional)

Create deployment apps directory:

```bash
sudo mkdir -p /opt/splunk/etc/deployment-apps
```

Create a basic app for Windows endpoints:

```bash
sudo mkdir -p /opt/splunk/etc/deployment-apps/windows_inputs/local
sudo nano /opt/splunk/etc/deployment-apps/windows_inputs/local/inputs.conf
```

Add this content:

```ini
[WinEventLog://Security]
disabled = false
index = security
renderXml = true

[WinEventLog://System]
disabled = false
index = windows

[WinEventLog://Application]
disabled = false
index = windows

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = sysmon
renderXml = true
```

Set permissions:

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/deployment-apps/windows_inputs
```

Reload deployment server:

```bash
sudo -u splunk /opt/splunk/bin/splunk reload deploy-server -auth admin:admin123!
```

## Verification Steps

### Check Receiving Port

```bash
sudo netstat -tuln | grep 9997
```

You should see Splunk listening on port 9997.

### Check Indexes

In Splunk Web:
- Go to **Settings > Indexes**
- Verify all indexes are created

### Check Installed Apps

- Go to **Apps > Manage Apps**
- Verify Sysmon and Windows add-ons are installed

### Test Search

Run this search to verify setup:

```spl
| eventcount summarize=false index=* 
| dedup index 
| fields index
```

## Useful Commands

### Check Splunk Status

```bash
sudo -u splunk /opt/splunk/bin/splunk status
```

### Restart Splunk

```bash
sudo -u splunk /opt/splunk/bin/splunk restart
```

### View Deployment Clients

```bash
sudo -u splunk /opt/splunk/bin/splunk list deploy-clients -auth admin:admin123!
```

### Check Listening Ports

```bash
sudo netstat -tulpn | grep splunk
```

## Troubleshooting

### Splunk Won't Start

```bash
# Check logs
sudo tail -f /opt/splunk/var/log/splunk/splunkd.log

# Check permissions
sudo chown -R splunk:splunk /opt/splunk
```

### Firewall Issues

```bash
# Check firewall status
sudo ufw status

# Allow additional ports if needed
sudo ufw allow 9997/tcp
sudo ufw reload
```

### Reset Admin Password

```bash
sudo -u splunk /opt/splunk/bin/splunk edit user admin -password NEW_PASSWORD -auth admin:OLD_PASSWORD
```

## Next Steps

1. Set up Universal Forwarders on endpoints
2. Configure deployment apps and server classes
3. Verify data is flowing into indexes
4. Create detection rules and dashboards

---

*Last updated: 2026-01-21*
