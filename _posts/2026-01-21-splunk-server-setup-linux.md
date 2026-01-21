---
layout: post
title: "Splunk Server Setup Guide - Linux"
date: 2026-01-21
categories: notes
tags: [splunk, linux, security, siem]
---

# Splunk Server Setup Guide - Linux Environment

Complete guide for setting up Splunk Server with deployment server configuration for receiving logs from Windows and Linux endpoints.

## Installation Script

### Configuration Variables

Edit these at the top before running:

#!/bin/bash
# ============================================
# CONFIGURATION - EDIT BEFORE RUNNING
# ============================================
DEPLOYMENT_SERVER_IP="167.172.73.162"     # Change to your Splunk server IP
SPLUNK_ADMIN="admin"                       # Admin username
SPLUNK_PASSWORD="admin123!"                # CHANGE to secure password
TIMEZONE="Asia/Kuala_Lumpur"               # Your timezone
# ============================================

### Full Installation Script

#!/bin/bash
set -e

# CONFIGURATION
DEPLOYMENT_SERVER_IP="167.172.73.162"
SPLUNK_ADMIN="admin"
SPLUNK_PASSWORD="admin123!"
TIMEZONE="Asia/Kuala_Lumpur"

SPLUNK_VERSION="10.0.2"
SPLUNK_BUILD="e2d18b4767e9"
SPLUNK_DEB="splunk-$SPLUNK_VERSION-$SPLUNK_BUILD-linux-amd64.deb"
SPLUNK_URL="https://download.splunk.com/products/splunk/releases/$SPLUNK_VERSION/linux/$SPLUNK_DEB"
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
sudo ufw allow 8000/tcp
sudo ufw allow 9997/tcp
sudo ufw allow 8089/tcp
sudo ufw reload

echo "[8/10] Creating splunk user..."
sudo useradd -m -s /bin/bash $SPLUNK_USER || true

echo "[9/10] Setting ownership..."
sudo $SPLUNK_HOME/bin/splunk stop
sudo chown -R $SPLUNK_USER:$SPLUNK_USER $SPLUNK_HOME
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk enable boot-start -user $SPLUNK_USER

echo "[10/10] Enabling deployment server..."
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk enable deploy-server -auth $SPLUNK_ADMIN:$SPLUNK_PASSWORD

sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk restart

echo ""
echo "Installation Complete!"
echo "Web Interface: http://YOUR_IP:8000"
echo "Username: $SPLUNK_ADMIN"
echo ""
echo "IMPORTANT: Complete the manual configuration steps below!"

## Post-Installation Manual Configuration

### Step 1: Enable Data Receiving (Port 9997)

You must enable the receiving port to accept data from forwarders.

#### Option A: Via Web Interface (Recommended)
1. Login to Splunk Web at http://YOUR_SERVER_IP:8000
2. Go to Settings > Forwarding and receiving
3. Click Configure receiving
4. Click New Receiving Port
5. Enter port: 9997
6. Click Save

#### Option B: Via Command Line
sudo -u splunk /opt/splunk/bin/splunk enable listen 9997 -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk restart

### Step 2: Create Indexes

Create dedicated indexes for different log types.

#### Via Web Interface
1. Go to Settings > Indexes
2. Click New Index
3. Create these indexes:

Index Name: sysmon
Max Size: 500GB (or as needed)
Click Save

Index Name: security  
Max Size: 500GB
Click Save

Index Name: windows
Max Size: 500GB
Click Save

Index Name: linux
Max Size: 500GB
Click Save

#### Via Command Line
sudo -u splunk /opt/splunk/bin/splunk add index sysmon -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk add index security -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk add index windows -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk add index linux -auth admin:admin123!

### Step 3: Install Splunk Add-ons

Install these apps to properly parse Windows and Sysmon logs.

#### Required Add-ons

1. Splunk Add-on for Microsoft Windows
   - Go to Apps > Find More Apps
   - Search for "Splunk Add-on for Microsoft Windows"
   - Click Install
   - This parses Windows Event Logs

2. Splunk Add-on for Sysmon
   - Search for "Splunk Add-on for Sysmon"
   - Click Install
   - This parses Sysmon logs with proper field extraction

3. Splunk Add-on for Unix and Linux (if monitoring Linux)
   - Search for "Splunk Add-on for Unix and Linux"
   - Click Install

After installation, restart Splunk:
sudo -u splunk /opt/splunk/bin/splunk restart

### Step 4: Configure Deployment Server (Optional)

If you want to centrally manage forwarder configurations:

1. Create deployment apps directory:
sudo mkdir -p /opt/splunk/etc/deployment-apps

2. Create a basic app for Windows endpoints:
sudo mkdir -p /opt/splunk/etc/deployment-apps/windows_inputs/local
sudo nano /opt/splunk/etc/deployment-apps/windows_inputs/local/inputs.conf

Add this content:

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

3. Reload deployment server:
sudo -u splunk /opt/splunk/bin/splunk reload deploy-server -auth admin:admin123!

## Verification Steps

### Check Receiving Port
sudo netstat -tuln | grep 9997

You should see Splunk listening on port 9997.

### Check Indexes
In Splunk Web, go to Settings > Indexes
Verify all indexes are created.

### Check Installed Apps
Go to Apps > Manage Apps
Verify the Sysmon and Windows add-ons are installed.

### Test Search
Run this search to verify setup:
| eventcount summarize=false index=* | dedup index | fields index

## Next Steps

1. Set up Universal Forwarders on endpoints
2. Configure Windows endpoints for enhanced logging
3. Verify data is flowing into indexes
4. Create detection rules and dashboards

---

Last updated: 2026-01-21
