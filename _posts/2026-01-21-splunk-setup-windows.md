---
layout: post
title: "Splunk Windows Endpoint & Forwarder Setup Guide"
date: 2026-01-21
categories: notes
tags: [splunk, windows, forwarder, sysmon, security]
---

# Splunk Windows Endpoint & Forwarder Setup Guide

This is a **complete end-to-end guide** for setting up a Windows machine as a Splunk-monitored endpoint. You will:

1. Enable advanced Windows logging (Event ID 4688, PowerShell)
2. Install and configure Sysmon
3. Install Splunk Universal Forwarder
4. Configure log collection and forwarding

Follow these steps **in order** on every Windows endpoint you want to monitor.

---

## Prerequisites

- Windows Server 2016 or later / Windows 10/11
- Administrator privileges
- PowerShell 5.1 or later
- Splunk Server already set up and running

---

## Part 1: Enable Windows Advanced Logging

Before anything else, enable enhanced logging so Windows captures detailed security events.

### Enable Process Creation Auditing (Event ID 4688)

Event ID 4688 records every process that starts on the system — essential for detecting malicious activity.

#### Via Group Policy (Recommended for Domain)

1. Open **Group Policy Management**
2. Edit your policy or create a new one
3. Navigate to:
   ```
   Computer Configuration
   └── Policies
       └── Windows Settings
           └── Security Settings
               └── Advanced Audit Policy Configuration
                   └── System Audit Policies
                       └── Detailed Tracking
   ```
4. Double-click **"Audit Process Creation"**
5. Check **"Configure the following audit events"**
6. Check **"Success"** → Click **OK**

7. Now enable Command Line logging. Navigate to:
   ```
   Computer Configuration
   └── Administrative Templates
       └── System
           └── Audit Process Creation
   ```
8. Double-click **"Include command line in process creation events"**
9. Select **"Enabled"** → Click **OK**

#### Via PowerShell (Quickest Method)

Run as Administrator:

```powershell
# Enable Process Creation Auditing
auditpol /set /subcategory:"Process Creation" /success:enable

# Enable Command Line logging in Event ID 4688
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" `
    /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f

# Verify it is enabled
auditpol /get /subcategory:"Process Creation"
```

Expected output:
```
System audit policy
Category/Subcategory                      Setting
Detailed Tracking
  Process Creation                        Success
```

### Enable PowerShell Script Block Logging

Captures every PowerShell command executed — crucial for detecting PowerShell-based attacks.

#### Via Group Policy

1. Navigate to:
   ```
   Computer Configuration
   └── Administrative Templates
       └── Windows Components
           └── Windows PowerShell
   ```
2. Double-click **"Turn on PowerShell Script Block Logging"**
3. Select **"Enabled"**
4. Optionally check **"Log script block invocation start/stop events"**
5. Click **OK**

#### Via PowerShell

```powershell
# Enable Script Block Logging
$sbPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $sbPath -Force | Out-Null
Set-ItemProperty -Path $sbPath -Name "EnableScriptBlockLogging" -Value 1 -Type DWord

# Optional: Enable invocation logging
Set-ItemProperty -Path $sbPath -Name "EnableScriptBlockInvocationLogging" -Value 1 -Type DWord

# Verify
Get-ItemProperty -Path $sbPath
```

Script Block logs appear in:
- **Event Log:** `Microsoft-Windows-PowerShell/Operational`
- **Event ID:** `4104`

### Enable PowerShell Module Logging (Optional)

Logs all PowerShell module activity for deeper visibility:

```powershell
# Enable Module Logging
$mlPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging"
New-Item -Path $mlPath -Force | Out-Null
Set-ItemProperty -Path $mlPath -Name "EnableModuleLogging" -Value 1 -Type DWord

# Log all modules
$mlNames = "$mlPath\ModuleNames"
New-Item -Path $mlNames -Force | Out-Null
Set-ItemProperty -Path $mlNames -Name "*" -Value "*"
```

---

## Part 2: Install Sysmon

Sysmon (System Monitor) provides rich, detailed logging of system activity that standard Windows logs miss — process creation with hashes, network connections, file creation, and more.

### Download and Install Sysmon

Run as Administrator:

```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
    -OutFile "$env:TEMP\Sysmon.zip"

# Extract
Expand-Archive -Path "$env:TEMP\Sysmon.zip" -DestinationPath "$env:TEMP\Sysmon" -Force

# Download SwiftOnSecurity recommended config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
    -OutFile "$env:TEMP\sysmonconfig.xml"

# Install Sysmon with config
Start-Process -FilePath "$env:TEMP\Sysmon\Sysmon64.exe" `
    -ArgumentList "-accepteula","-i","$env:TEMP\sysmonconfig.xml" `
    -Wait

# Verify Sysmon is running
Get-Service Sysmon64
```

Expected output:
```
Status   Name       DisplayName
------   ----       -----------
Running  Sysmon64   Sysmon64
```

Sysmon logs appear in:
- **Event Log:** `Microsoft-Windows-Sysmon/Operational`

### Update Sysmon Config Later

```powershell
# Update to a new config
Start-Process -FilePath "C:\Windows\Sysmon64.exe" `
    -ArgumentList "-c","C:\path\to\new-config.xml" -Wait
```

### Key Sysmon Event IDs

| Event ID | Description |
|----------|-------------|
| 1 | Process Creation |
| 3 | Network Connection |
| 5 | Process Terminated |
| 7 | Image/DLL Loaded |
| 8 | CreateRemoteThread |
| 10 | Process Access |
| 11 | File Created |
| 12/13 | Registry Create/Set |
| 22 | DNS Query |

---

## Part 3: Install Splunk Universal Forwarder

### Configuration Variables

Edit these before running the script:

```powershell
$SplunkServer    = "167.172.73.162"       # Your Splunk server IP
$ReceivingPort   = "9997"                  # Splunk receiving port
$DeploymentServer = "167.172.73.162:8089"  # Deployment server
$AdminPassword   = "changeme"              # Forwarder admin password
```

### Method 1: Silent Installation (Automated)

Run as Administrator:

```powershell
# ============================================
# CONFIGURATION - EDIT THESE
# ============================================
$SplunkServer     = "167.172.73.162"
$ReceivingPort    = "9997"
$DeploymentServer = "167.172.73.162:8089"
$AdminPassword    = "changeme"
# ============================================

Write-Host "=== Splunk Universal Forwarder Installation ===" -ForegroundColor Green

# Step 1: Download
Write-Host "[1/4] Downloading Universal Forwarder..." -ForegroundColor Yellow
$ufUrl = "https://download.splunk.com/products/universalforwarder/releases/9.3.2/windows/splunkforwarder-9.3.2-d8bb32809498-x64-release.msi"
$msiPath = "$env:TEMP\splunk-forwarder.msi"
Invoke-WebRequest -Uri $ufUrl -OutFile $msiPath

# Step 2: Install
Write-Host "[2/4] Installing..." -ForegroundColor Yellow
$receivingIndexer = "$SplunkServer" + ":" + "$ReceivingPort"
$arguments = "/i `"$msiPath`" AGREETOLICENSE=Yes SPLUNKUSERNAME=admin SPLUNKPASSWORD=$AdminPassword RECEIVING_INDEXER=$receivingIndexer DEPLOYMENT_SERVER=$DeploymentServer /quiet"
Start-Process msiexec.exe -ArgumentList $arguments -Wait -NoNewWindow

# Step 3: Verify
Write-Host "[3/4] Verifying..." -ForegroundColor Yellow
Start-Sleep -Seconds 5
$service = Get-Service SplunkForwarder -ErrorAction SilentlyContinue

if ($service) {
    Write-Host "Service Status: $($service.Status)" -ForegroundColor Cyan
} else {
    Write-Host "Service not found - check installation logs!" -ForegroundColor Red
    exit 1
}

# Step 4: Cleanup
Write-Host "[4/4] Cleaning up..." -ForegroundColor Yellow
Remove-Item $msiPath -Force

Write-Host ""
Write-Host "=== Installation Complete! ===" -ForegroundColor Green
Write-Host "Splunk Server  : $SplunkServer`:$ReceivingPort" -ForegroundColor Cyan
Write-Host "Deploy Server  : $DeploymentServer" -ForegroundColor Cyan
```

### Method 2: GUI Installation (Manual)

1. Download from [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)
2. Run MSI as Administrator
3. Accept the license agreement
4. Set admin credentials
5. Configure:
   - **Receiving Indexer:** `YOUR_SPLUNK_IP:9997`
   - **Deployment Server:** `YOUR_SPLUNK_IP:8089`
6. Complete installation

---

## Part 4: Configure Log Collection

After installation, configure which logs to collect and forward.

### Create inputs.conf

Open notepad as Administrator and create/edit:
```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

```ini
[default]
host = $decideOnStartup

# -----------------------------------------------
# Windows Security Log
# Captures: Logon, Account, Process Creation (4688)
# -----------------------------------------------
[WinEventLog://Security]
disabled = false
index = security
renderXml = true

# -----------------------------------------------
# Windows System Log
# Captures: Service changes, driver issues, errors
# -----------------------------------------------
[WinEventLog://System]
disabled = false
index = windows
renderXml = true

# -----------------------------------------------
# Windows Application Log
# Captures: App crashes, .NET errors
# -----------------------------------------------
[WinEventLog://Application]
disabled = false
index = windows
renderXml = true

# -----------------------------------------------
# Sysmon Log - CRITICAL for security monitoring
# Captures: Process, network, file, registry activity
# -----------------------------------------------
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = sysmon
renderXml = true

# -----------------------------------------------
# PowerShell Script Block Logging
# Captures: All executed PowerShell commands (Event ID 4104)
# -----------------------------------------------
[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = false
index = security
renderXml = true

# -----------------------------------------------
# PowerShell Classic Log
# -----------------------------------------------
[WinEventLog://Windows PowerShell]
disabled = false
index = security
renderXml = true
```

### Create outputs.conf (if not configured during install)

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
```

```ini
[tcpout]
defaultGroup = splunk_indexers

[tcpout:splunk_indexers]
server = 167.172.73.162:9997
```

### Restart Forwarder to Apply Changes

```powershell
Restart-Service SplunkForwarder
```

---

## Part 5: Verification

### Check Service Status

```powershell
Get-Service SplunkForwarder
```

### Check Connection to Splunk Server

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"
.\splunk.exe list forward-server
```

Expected: `Active forwards: 167.172.73.162:9997`

### Check Deployment Server Connection

```powershell
.\splunk.exe show deploy-poll
```

### Test Network Connectivity

```powershell
# Test receiving port
Test-NetConnection -ComputerName 167.172.73.162 -Port 9997

# Test deployment server port
Test-NetConnection -ComputerName 167.172.73.162 -Port 8089
```

Both should show: `TcpTestSucceeded : True`

### View Forwarder Logs

```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50
```

---

## Part 6: Verify Data in Splunk Web

Login to Splunk Web and run these searches:

### Check All Data from This Host

```spl
index=* host=YOUR_WINDOWS_HOSTNAME
| stats count by index, sourcetype
```

### Check Process Creation Events (4688)

```spl
index=security EventCode=4688
| table _time, host, NewProcessName, CommandLine
| head 20
```

If `CommandLine` is populated, command line logging is working!

### Check Sysmon Events

```spl
index=sysmon
| stats count by EventCode
| sort - count
```

### Check PowerShell Script Block Logs

```spl
index=security EventCode=4104
| table _time, host, ScriptBlockText
| head 10
```

### Check All Windows Security Events

```spl
index=security sourcetype="WinEventLog:Security"
| stats count by EventCode
| sort - count
```

---

## Troubleshooting

### No Data in Splunk

```powershell
# 1. Check service is running
Get-Service SplunkForwarder

# 2. Check connection
cd "C:\Program Files\SplunkUniversalForwarder\bin"
.\splunk.exe list forward-server

# 3. Check firewall
Test-NetConnection -ComputerName 167.172.73.162 -Port 9997

# 4. Check inputs
.\splunk.exe btool inputs list --debug
```

### Sysmon Events Not Appearing

```powershell
# Verify Sysmon is running
Get-Service Sysmon64

# Check Sysmon logs exist locally
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

Ensure `renderXml = true` in inputs.conf and install **Splunk Add-on for Sysmon** on the Splunk server.

### Event ID 4688 Not Showing Command Line

```powershell
# Check audit policy
auditpol /get /subcategory:"Process Creation"

# Check registry key
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit"
```

> **Note:** A reboot is sometimes required after enabling command line logging.

### Service Won't Start

```powershell
# Check Windows Event Viewer for errors
Get-EventLog -LogName Application -Source Splunk* -Newest 20

# Check Splunk own logs
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 100
```

---

## Summary Checklist

- [ ] Enabled Process Creation Auditing (Event ID 4688)
- [ ] Enabled Command Line in Process Events
- [ ] Enabled PowerShell Script Block Logging
- [ ] Installed Sysmon with SwiftOnSecurity config
- [ ] Installed Splunk Universal Forwarder
- [ ] Configured inputs.conf for all log sources
- [ ] Verified service is running
- [ ] Tested connectivity to Splunk server (9997) and deployment server (8089)
- [ ] Verified data appearing in Splunk Web
- [ ] Installed Sysmon and Windows add-ons on Splunk server

---

## Next Steps

1. [Set up Deployment Server for centralized management]({% post_url 2026-01-21-splunk-deployment-server %})
2. Configure Server Classes to push configs automatically
3. Create detection rules and alerts in Splunk
4. Build dashboards to visualize endpoint activity

---

*Last updated: 2026-01-21*
