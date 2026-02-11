---
layout: post
title: "Splunk Deployment Server and Server Classes Guide"
date: 2026-01-21
categories: [notes, splunk]
tags: [splunk, deployment, management]
---

# Splunk Deployment Server Management

Complete guide to setting up Deployment Server, creating Apps, and managing Server Classes to push configurations to forwarders centrally.

## What is Deployment Server?

Deployment Server allows you to centrally manage configurations for multiple Universal Forwarders. Instead of manually configuring each forwarder, you create apps on the Deployment Server and push them to groups of forwarders using Server Classes.

## Architecture Overview

```
Deployment Server (Port 8089)
       |
       | Deploy Apps
       v
Server Classes (Groups)
       |
       | Assign to
       v
Forwarders (Deployment Clients)
```

## Step-by-Step Setup

### Step 1: Enable Deployment Server

On your Splunk Server:

```bash
sudo -u splunk /opt/splunk/bin/splunk enable deploy-server -auth admin:admin123!
sudo -u splunk /opt/splunk/bin/splunk restart
```

Verify it is enabled:

```bash
sudo -u splunk /opt/splunk/bin/splunk show deploy-server
```

### Step 2: Access Forwarder Management

1. Login to Splunk Web at `http://YOUR_SERVER:8000`
2. Go to **Settings > Forwarder management**

You will see three tabs:
- **Forwarders** - Shows all connected forwarders
- **Apps** - Configuration packages to deploy
- **Server Classes** - Groups of forwarders that receive specific apps

### Step 3: Create a Deployment App for Windows

Apps are configuration packages stored in:
```
/opt/splunk/etc/deployment-apps/
```

#### Create Windows Inputs App

```bash
sudo mkdir -p /opt/splunk/etc/deployment-apps/windows_inputs/local
sudo mkdir -p /opt/splunk/etc/deployment-apps/windows_inputs/metadata
```

Create `app.conf`:

```bash
sudo nano /opt/splunk/etc/deployment-apps/windows_inputs/local/app.conf
```

Add this content:

```ini
[install]
state = enabled

[ui]
is_visible = false
is_manageable = false

[launcher]
author = Admin
version = 1.0
description = Windows event log collection
```

Create `inputs.conf`:

```bash
sudo nano /opt/splunk/etc/deployment-apps/windows_inputs/local/inputs.conf
```

Add this content:

```ini
[default]
host = $decideOnStartup

[WinEventLog://Security]
disabled = false
index = security
renderXml = true

[WinEventLog://System]
disabled = false
index = windows
renderXml = true

[WinEventLog://Application]
disabled = false
index = windows
renderXml = true

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = sysmon
renderXml = true

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = false
index = security
renderXml = true
```

Create `default.meta`:

```bash
sudo nano /opt/splunk/etc/deployment-apps/windows_inputs/metadata/default.meta
```

Add this content:

```ini
[]
access = read : [ * ], write : [ admin ]
export = system
```

Set permissions:

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/deployment-apps/windows_inputs
```

### Step 4: Create a Deployment App for Linux

```bash
sudo mkdir -p /opt/splunk/etc/deployment-apps/linux_inputs/local
sudo mkdir -p /opt/splunk/etc/deployment-apps/linux_inputs/metadata
```

Create `app.conf`:

```bash
sudo nano /opt/splunk/etc/deployment-apps/linux_inputs/local/app.conf
```

```ini
[install]
state = enabled

[ui]
is_visible = false
is_manageable = false

[launcher]
author = Admin
version = 1.0
description = Linux system log collection
```

Create `inputs.conf`:

```bash
sudo nano /opt/splunk/etc/deployment-apps/linux_inputs/local/inputs.conf
```

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

Create `default.meta` and set permissions:

```bash
sudo nano /opt/splunk/etc/deployment-apps/linux_inputs/metadata/default.meta
```

```ini
[]
access = read : [ * ], write : [ admin ]
export = system
```

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/deployment-apps/linux_inputs
```

### Step 5: Reload Deployment Server

After creating apps, reload the deployment server:

```bash
sudo -u splunk /opt/splunk/bin/splunk reload deploy-server -auth admin:admin123!
```

Or via Web UI:
1. **Settings > Forwarder management**
2. **Apps** tab
3. Click **Reload**

### Step 6: Create Server Classes via Web UI

#### Create Windows Server Class

1. Go to **Settings > Forwarder management**
2. Click **Server Classes** tab
3. Click **New Server Class**

**Server Class Configuration:**
- **Name:** `windows_servers`
- **Description:** `All Windows endpoints`
- Click **Save**

In the Server Class detail page:

**Add Apps:**
1. Click **"Add Apps"**
2. Select **"windows_inputs"**
3. Click **Save**

**Add Clients** (Choose one method):

**Method A - Include by hostname:**
1. Click **"Edit"** next to Clients
2. In **"Include"** section, add pattern: `*windows*` or `*WIN*`
3. Click **Save**

**Method B - Include by IP:**
1. In **"Include"** section, add pattern: `192.168.1.*`
2. Click **Save**

**Method C - Include all, exclude specific:**
1. In **"Include"** section, add: `*`
2. In **"Exclude"** section, add patterns for non-Windows hosts
3. Click **Save**

Click **"Enable"** to activate the Server Class

#### Create Linux Server Class

1. Click **"New Server Class"**
2. **Name:** `linux_servers`
3. **Description:** `All Linux endpoints`
4. Click **Save**

**Add Apps:**
1. Click **"Add Apps"**
2. Select **"linux_inputs"**
3. Click **Save**

**Add Clients:**
1. Click **"Edit"** next to Clients
2. Include pattern: `*linux*` or `*ubuntu*` or `*centos*`
3. Click **Save**

Click **"Enable"** to activate

### Step 7: Verify Apps in Web UI

Go to **Settings > Forwarder management > Apps**

You should see:
- `windows_inputs`
- `linux_inputs`

Click on each app to see:
- Which Server Classes use it
- Configuration files included

### Step 8: Check Forwarders Connect

After forwarders connect (configured with deployment server), check:

**Settings > Forwarder management > Forwarders**

You should see:
- Hostname
- IP Address
- Server Class assigned
- Apps deployed
- Last phone home time

### Step 9: Verify on Forwarder

**On Windows forwarder:**

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"
.\splunk.exe list deploy-client
```

**On Linux forwarder:**

```bash
sudo /opt/splunkforwarder/bin/splunk list deploy-client
```

Should show:
- Deployment server connected
- Apps downloaded
- Last check-in time

**Check deployed apps on forwarder:**

Windows:
```powershell
dir "C:\Program Files\SplunkUniversalForwarder\etc\apps"
```

Linux:
```bash
ls -la /opt/splunkforwarder/etc/apps/
```

You should see `windows_inputs` or `linux_inputs` folder

### Step 10: Force Update (if needed)

To force forwarders to check for updates immediately:

**On forwarder:**

Windows:
```powershell
.\splunk.exe reload deploy-client
```

Linux:
```bash
sudo /opt/splunkforwarder/bin/splunk reload deploy-client
```

Or restart the forwarder service.

## Advanced: Create Server Class via CLI

Instead of Web UI, you can edit `serverclass.conf`:

```bash
sudo nano /opt/splunk/etc/system/local/serverclass.conf
```

Example configuration:

```ini
[serverClass:windows_servers]
whitelist.0 = *windows*
whitelist.1 = *WIN*

[serverClass:windows_servers:app:windows_inputs]
restartSplunkd = true
stateOnClient = enabled

[serverClass:linux_servers]
whitelist.0 = *linux*
whitelist.1 = *ubuntu*

[serverClass:linux_servers:app:linux_inputs]
restartSplunkd = true
stateOnClient = enabled
```

Reload deployment server:

```bash
sudo -u splunk /opt/splunk/bin/splunk reload deploy-server -auth admin:admin123!
```

## Troubleshooting

### Forwarder Not Showing Up

1. **Check forwarder is configured with deployment server:**
   - `deploymentclient.conf` should point to your server:8089

2. **Check firewall allows port 8089**

3. **Check forwarder logs:**
   - Windows: `C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log`
   - Linux: `/opt/splunkforwarder/var/log/splunk/splunkd.log`

Search for "DeploymentClient" in logs

### App Not Deploying

1. Verify app exists in `/opt/splunk/etc/deployment-apps/`

2. Check app is added to Server Class

3. Check Server Class is **Enabled**

4. Verify forwarder matches whitelist pattern

5. Reload deployment server:
   ```bash
   sudo -u splunk /opt/splunk/bin/splunk reload deploy-server
   ```

6. Force forwarder to check:
   ```bash
   # On forwarder
   splunk reload deploy-client
   ```

### App Deployed but Not Working

1. **Check app permissions:**
   ```bash
   sudo ls -la /opt/splunk/etc/deployment-apps/
   ```

2. **Verify inputs.conf syntax:**
   ```bash
   sudo /opt/splunk/bin/splunk btool inputs list --debug
   ```

3. **Check forwarder received and enabled the app:**
   - On forwarder, check `/opt/splunkforwarder/etc/apps/`

4. **Restart forwarder** to ensure config is loaded

## Best Practices

1. **Name apps clearly:** `environment_purpose` (e.g., `windows_inputs`, `linux_monitoring`)

2. **Use Server Classes to group by:**
   - Operating System (Windows, Linux)
   - Environment (Production, Development)
   - Location (DataCenter1, DataCenter2)
   - Function (WebServers, DatabaseServers)

3. **Test apps on one forwarder** before deploying to Server Class

4. **Version your apps** in app.conf

5. **Document whitelist/blacklist patterns**

6. **Use `restartSplunkd = true`** in Server Class for input changes

7. **Monitor deployment server logs** for errors

8. **Regularly check Forwarder management** for disconnected clients

## Summary Workflow

1. Enable Deployment Server
2. Create app in `/opt/splunk/etc/deployment-apps/`
3. Reload Deployment Server
4. Create Server Class via Web UI
5. Add app to Server Class
6. Define client whitelist patterns
7. Enable Server Class
8. Forwarders auto-download app on next check-in
9. Verify in Forwarder management
10. Check data flowing in Splunk searches

## Example: Complete Deployment Flow

```bash
# 1. Create app
sudo mkdir -p /opt/splunk/etc/deployment-apps/my_app/local

# 2. Add configuration
sudo nano /opt/splunk/etc/deployment-apps/my_app/local/inputs.conf

# 3. Set permissions
sudo chown -R splunk:splunk /opt/splunk/etc/deployment-apps/my_app

# 4. Reload deployment server
sudo -u splunk /opt/splunk/bin/splunk reload deploy-server

# 5. Create Server Class in Web UI
# 6. Add app to Server Class
# 7. Define clients (whitelist)
# 8. Enable Server Class

# 9. Verify on forwarder
sudo /opt/splunkforwarder/bin/splunk list deploy-client
```

---

*Last updated: 2026-01-21*
