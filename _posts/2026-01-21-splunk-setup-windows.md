---
layout: post
title: "Splunk Setup Guide - Windows"
date: 2026-01-21
categories: notes
tags: [splunk, windows]
---

# Splunk Setup - Windows Environment

## Splunk Enterprise Installation

### Download

Visit Splunk website or use PowerShell:

$url = "https://download.splunk.com/products/splunk/releases/10.0.2/windows/splunk-10.0.2-e2d18b4767e9-x64-release.msi"
$output = "$env:TEMP\splunk.msi"
Invoke-WebRequest -Uri $url -OutFile $output

### GUI Installation Steps

1. Run MSI installer
2. Accept license
3. Choose installation directory
4. Create admin account
5. Configure deployment (optional)
6. Complete installation

### Configure Firewall

New-NetFirewallRule -DisplayName "Splunk Web" -LocalPort 8000 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Splunk Receiver" -LocalPort 9997 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Splunk Management" -LocalPort 8089 -Protocol TCP -Action Allow

### Enable Receiving

cd "C:\Program Files\Splunk\bin"
.\splunk.exe enable listen 9997 -auth admin:YourPassword

## Universal Forwarder Installation

### Silent Installation

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

### Configure inputs.conf

[WinEventLog://Security]
disabled = false
index = security

[WinEventLog://System]
disabled = false
index = windows

### Restart Service

Restart-Service SplunkForwarder
