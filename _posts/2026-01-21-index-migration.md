---
layout: post
title: "Splunk Blue Team Series: Index Migration, Recovery & Internals (Sysmon & Security)"
date: 2026-01-22
categories: notes
tags: [splunk, blueteam, dfir, sysmon, security, index, recovery]
---

# Splunk Blue Team Series  
## Index Migration, Recovery & Internals (Sysmon & Security)

This post is part of the **Splunk Blue Team Series**, focusing on **data survivability** after detection is already in place.

It documents **real-world Splunk recovery scenarios** commonly faced by SOC and DFIR teams, especially after:
- Server rebuilds
- Accidental reinstallation
- Lab or attack simulation environments
- Manual index folder recovery

---

## Series Context

**Previous post:**  
- Windows Endpoint Telemetry with Splunk (4688, PowerShell, Sysmon, Forwarder)

**This post focuses on:**  
- Migrating Splunk indexes safely  
- Recovering `sysmon` and `security` data  
- Avoiding silent data loss  
- Understanding why `fsck` is mandatory  

---

## What You Are Migrating

You are migrating **raw index buckets**, not apps or configs.

Splunk index path:
/opt/splunk/var/lib/splunk/

Example index folders:
sysmon/
security/

Each index contains:
- `db/` – hot & warm buckets
- `colddb/` – cold buckets
- `thaweddb/` – restored archives (if any)

---

## CRITICAL RULES (Do Not Skip)

⚠️ Splunk **must be stopped** before touching index folders  
⚠️ Ownership **must be `splunk:splunk`**  
⚠️ Indexes **must exist before restoring data**  
⚠️ Always run Splunk commands as the **splunk user**

Most failed recoveries are caused by ignoring one of these.

---

## Scenario 1: Archive-Based Migration (Recommended)

Use this if:
- Splunk A is still running
- You want a clean backup
- Index size is large

### On Splunk A (Source)

```bash
sudo /opt/splunk/bin/splunk stop
cd /opt/splunk/var/lib/splunk/

sudo tar -czf /tmp/sysmon_backup.tar.gz sysmon/
sudo tar -czf /tmp/security_backup.tar.gz security/

sudo /opt/splunk/bin/splunk start
Transfer to Splunk B:
scp /tmp/sysmon_backup.tar.gz root@splunkB_ip:/tmp/
scp /tmp/security_backup.tar.gz root@splunkB_ip:/tmp/
On Splunk B (Destination)
sudo -u splunk /opt/splunk/bin/splunk add index sysmon -auth admin:splunkadmin123!
sudo -u splunk /opt/splunk/bin/splunk add index security -auth admin:splunkadmin123!

sudo /opt/splunk/bin/splunk stop

sudo rm -rf /opt/splunk/var/lib/splunk/sysmon/
sudo rm -rf /opt/splunk/var/lib/splunk/security/

cd /opt/splunk/var/lib/splunk/
sudo tar -xzf /tmp/sysmon_backup.tar.gz
sudo tar -xzf /tmp/security_backup.tar.gz

sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/sysmon/
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/security/

sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=sysmon
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=security

sudo /opt/splunk/bin/splunk start
Scenario 2: Direct Folder Copy (No Tar)
Use this if you already copied the folders manually (scp, WinSCP, rsync, FileZilla).
Yes — this method is 100% valid.

sudo -u splunk /opt/splunk/bin/splunk add index sysmon -auth admin:splunkadmin123!
sudo -u splunk /opt/splunk/bin/splunk add index security -auth admin:splunkadmin123!

sudo /opt/splunk/bin/splunk stop

sudo rm -rf /opt/splunk/var/lib/splunk/sysmon/
sudo rm -rf /opt/splunk/var/lib/splunk/security/

sudo mv /tmp/sysmon /opt/splunk/var/lib/splunk/
sudo mv /tmp/security /opt/splunk/var/lib/splunk/

sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/sysmon/
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/security/

sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=sysmon
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=security

sudo /opt/splunk/bin/splunk start
Verification
CLI
sudo -u splunk /opt/splunk/bin/splunk search "index=sysmon | stats count" -auth admin:splunkadmin123!
sudo -u splunk /opt/splunk/bin/splunk search "index=security | stats count" -auth admin:splunkadmin123!
Web UI
index=sysmon | stats count by sourcetype
index=security | stats count by sourcetype
Common Errors & Fixes
Index Exists but No Data
Fix:
splunk fsck repair --all-buckets-one-index --index-name=sysmon
fsck Permission Errors
Fix:
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/sysmon/
Searches Slow or Incomplete
Fix:
Stop Splunk → run fsck → start Splunk
Single Reusable Migration Script
Save as migrate_indexes.sh
#!/bin/bash

SPLUNK_HOME="/opt/splunk"
INDEX_PATH="$SPLUNK_HOME/var/lib/splunk"
INDEXES=("sysmon" "security")
AUTH="admin:splunkadmin123!"

sudo $SPLUNK_HOME/bin/splunk stop

for index in "${INDEXES[@]}"; do
  sudo -u splunk $SPLUNK_HOME/bin/splunk add index $index -auth $AUTH
  sudo rm -rf $INDEX_PATH/$index
  sudo mv /tmp/$index $INDEX_PATH/
done

sudo chown -R splunk:splunk $INDEX_PATH

for index in "${INDEXES[@]}"; do
  sudo -u splunk $SPLUNK_HOME/bin/splunk fsck repair \
    --all-buckets-one-index --index-name=$index
done

sudo $SPLUNK_HOME/bin/splunk start
Why fsck Is Required (Splunk Internals)
Splunk does not auto-discover copied index data.
It relies on:

Bucket metadata
Time range registration
Journal state
When index folders are copied:
Raw data exists ✅
Splunk metadata does not ❌
fsck repair:
Scans all buckets
Rebuilds metadata
Re-registers time ranges
Makes data searchable again
Without fsck:
Logs silently disappear
DFIR timelines break
Attacks become invisible
Simplified Timeline
1. Install Splunk
2. Create indexes
3. Stop Splunk
4. Copy index folders
5. Fix ownership
6. Run fsck
7. Start Splunk
8. Verify data
