---
layout: post
title: "Splunk Universal Forwarder Setup - Linux"
date: 2026-01-21
categories: notes
tags: [splunk, linux, forwarder]
---

# Splunk Universal Forwarder Setup - Linux

Complete guide for installing Splunk Universal Forwarder on Linux endpoints.

## Configuration Variables

Edit these at the top of the script:

```bash
DEPLOYMENT_SERVER="167.172.73.162:8089"
SPLUNK_RECEIVER="167.172.73.162:9997"
TIMEZONE="Asia/Kuala_Lumpur"
```

## Installation Script

```bash
#!/bin/bash
set -e

# ============================================
# CONFIGURATION - EDIT THESE
# ============================================
DEPLOYMENT_SERVER="167.172.73.162:8089"
SPLUNK_RECEIVER="167.172.73.162:9997"
TIMEZONE="Asia/Kuala_Lumpur"
# ============================================

SPLUNK_VERSION="9.3.2"
SPLUNK_BUILD="d8bb32809498"
UF_DEB="splunkforwarder-${SPLUNK_VERSION}-${SPLUNK_BUILD}-linux-2.6-amd64.deb"
UF_URL="https://download.splunk.com/products/universalforwarder/releases/${SPLUNK_VERSION}/linux/${UF_DEB}"
SPLUNK_HOME="/opt/splunkforwarder"

echo "=== Universal Forwarder Installation ==="

echo "[1/6] Setting timezone..."
sudo timedatectl set-timezone $TIMEZONE

echo "[2/6] Updating system..."
sudo apt update && sudo apt upgrade -y

echo "[3/6] Downloading Universal Forwarder..."
cd /tmp
wget -O $UF_DEB "$UF_URL"

echo "[4/6] Installing..."
sudo dpkg -i $UF_DEB
rm $UF_DEB

echo "[5/6] Configuring..."
sudo $SPLUNK_HOME/bin/splunk start --accept-license --answer-yes
sudo $SPLUNK_HOME/bin/splunk set deploy-poll $DEPLOYMENT_SERVER -auth admin:changeme
sudo $SPLUNK_HOME/bin/splunk add forward-server $SPLUNK_RECEIVER -auth admin:changeme

echo "[6/6] Enabling boot-start..."
sudo $SPLUNK_HOME/bin/splunk enable boot-start
sudo $SPLUNK_HOME/bin/splunk restart

echo ""
echo "Installation Complete!"
echo "Deployment Server: $DEPLOYMENT_SERVER"
echo "Forward Server: $SPLUNK_RECEIVER"
```

## Verification

Check forwarder status:

```bash
sudo /opt/splunkforwarder/bin/splunk status
```

Check deployment server connection:

```bash
sudo /opt/splunkforwarder/bin/splunk show deploy-poll
```

Check forwarding configuration:

```bash
sudo /opt/splunkforwarder/bin/splunk list forward-server
```

## Manual Configuration (Optional)

If you need to manually configure inputs, edit:

```bash
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```

Example Linux inputs:

```ini
[default]
host = $decideOnStartup

[monitor:///var/log/syslog]
disabled = false
index = linux
sourcetype = syslog

[monitor:///var/log/auth.log]
disabled = false
index = security
sourcetype = linux_secure

[monitor:///var/log/secure]
disabled = false
index = security
sourcetype = linux_secure
```

Restart after changes:

```bash
sudo /opt/splunkforwarder/bin/splunk restart
```

---

*Last updated: 2026-01-21*
