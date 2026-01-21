---
layout: post
title: "Splunk Setup Guide – Windows"
date: 2026-01-21
categories: notes
tags: [splunk, windows, setup]
---

# Splunk Setup Guide – Windows  
*Date: Jan 21, 2026*  

This guide walks you through installing **Splunk Enterprise** and **Splunk Universal Forwarder** on Windows, configuring firewall rules, enabling receiving, and forwarding logs to your Splunk server.  

---

## 1️⃣ Splunk Enterprise Installation

### Download Splunk
You can download manually from the [Splunk website](https://www.splunk.com/en_us/download.html) or use PowerShell:

$url = "https://download.splunk.com/products/splunk/releases/10.0.2/windows/splunk-10.0.2-e2d18b4767e9-x64-release.msi"
$output = "$env:TEMP\splunk.msi"
Invoke-WebRequest -Uri $url -OutFile $output
Explanation: Downloads the Splunk MSI installer to your temporary folder.
GUI Installation Steps
Run the downloaded MSI installer
Accept the license agreement
Choose the installation directory
Create an admin account (remember username & password)
Optionally configure deployment settings
Complete the installation
Tip: Splunk Web is usually available at http://localhost:8000.
Configure Firewall
New-NetFirewallRule -DisplayName "Splunk Web" -LocalPort 8000 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Splunk Receiver" -LocalPort 9997 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Splunk Management" -LocalPort 8089 -Protocol TCP -Action Allow
Explanation:
8000 → Splunk Web interface
9997 → Forwarder data receiver
8089 → Management / Deployment server port
Enable Receiving
cd "C:\Program Files\Splunk\bin"
.\splunk.exe enable listen 9997 -auth admin:YourPassword
Explanation: Enables Splunk to receive data from forwarders. Replace YourPassword with your admin password.
2️⃣ Splunk Universal Forwarder Installation
Silent Installation
$SPLUNK_SERVER = "167.172.73.162"
$RECEIVING_PORT = "9997"
$DEPLOYMENT_SERVER = "167.172.73.162:8089"

$msiPath = "$env:TEMP\splunk-forwarder.msi"
$arguments = @(
    "/i", $msiPath,
    "AGREETOLICENSE=Yes",
    "RECEIVING_INDEXER=$SPLUNK_SERVER:$RECEIVING_PORT",
    "DEPLOYMENT_SERVER=$DEPLOYMENT_SERVER",
    "/quiet"
)
Start-Process msiexec.exe -ArgumentList $arguments -Wait
Explanation: Installs the Universal Forwarder silently and points it to your Splunk server.
Configure inputs.conf
[WinEventLog://Security]
disabled = false
index = security

[WinEventLog://System]
disabled = false
index = windows
Explanation:
Collects Security and System Windows event logs. Security logs go to the security index; System logs go to the windows index.
Restart Splunk Forwarder
Restart-Service SplunkForwarder
Explanation: Restarts the service to apply configuration changes.
✅ Notes / Tips
Ensure firewall rules allow your Splunk server to receive data
Replace 167.172.73.162 with your actual Splunk server IP
Keep your admin credentials secure; do not put them in public repos
