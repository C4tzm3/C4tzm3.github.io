---
layout: post
title: "Index Migration, Recovery & Internals"
date: 2026-02-10
categories: [notes, splunk]
tags: [splunk, blueteam, dfir, sysmon, security, index, recovery]
---

# Index Migration, Recovery & Internals 

This post is part of the **Splunk Blue Team Series**, focusing on **data survivability** after detection is already in place.

It documents **real-world Splunk recovery scenarios** commonly faced by SOC and DFIR teams, especially after:
- Server rebuilds
- Accidental reinstallation
- Lab or attack simulation environments
- Manual index folder recovery

---

## Series Context

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

- Splunk **must be stopped** before touching index folders  
- Ownership **must be `splunk:splunk`**
- Indexes **must exist before restoring data**
- Always run Splunk commands as the **splunk user**

Most failed recoveries are caused by ignoring one of these.

---

## Scenario 1: Archive-Based Migration (Recommended)

Use this if:
- Splunk A is still running
- You want a clean backup
- Index size is large


## On Splunk A (Source)
```bash
# Stop Splunk
sudo /opt/splunk/bin/splunk stop

# Navigate to index directory
cd /opt/splunk/var/lib/splunk/

# Create backups
sudo tar -czf /tmp/sysmon_backup.tar.gz sysmon/
sudo tar -czf /tmp/security_backup.tar.gz security/

# Start Splunk again
sudo /opt/splunk/bin/splunk start
```
```bash
Transfer backups to Splunk B
scp /tmp/sysmon_backup.tar.gz root@splunkB_ip:/tmp/
scp /tmp/security_backup.tar.gz root@splunkB_ip:/tmp/
```
## On Splunk B (Destination)

```bash
# Create indexes first
sudo -u splunk /opt/splunk/bin/splunk add index sysmon -auth admin:splunkadmin123!
sudo -u splunk /opt/splunk/bin/splunk add index security -auth admin:splunkadmin123!

# Stop Splunk
sudo /opt/splunk/bin/splunk stop

# Remove empty index folders
sudo rm -rf /opt/splunk/var/lib/splunk/sysmon/
sudo rm -rf /opt/splunk/var/lib/splunk/security/

# Extract backups
cd /opt/splunk/var/lib/splunk/
sudo tar -xzf /tmp/sysmon_backup.tar.gz
sudo tar -xzf /tmp/security_backup.tar.gz

# Fix ownership (CRITICAL)
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/sysmon/
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/security/

# Repair indexes
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=sysmon
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=security

# Start Splunk
sudo /opt/splunk/bin/splunk start
```

## What if.. Direct Folder Copy (No Tar)?
Use this if you already copied the folders manually
(scp, WinSCP, rsync, FileZilla).

```bash
# Create indexes
sudo -u splunk /opt/splunk/bin/splunk add index sysmon -auth admin:splunkadmin123!
sudo -u splunk /opt/splunk/bin/splunk add index security -auth admin:splunkadmin123!

# Stop Splunk
sudo /opt/splunk/bin/splunk stop

# Remove empty folders
sudo rm -rf /opt/splunk/var/lib/splunk/sysmon/
sudo rm -rf /opt/splunk/var/lib/splunk/security/

# Move copied folders
sudo mv /tmp/sysmon /opt/splunk/var/lib/splunk/
sudo mv /tmp/security /opt/splunk/var/lib/splunk/

# Fix ownership
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/sysmon/
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/security/

# Repair indexes
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=sysmon
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=security

# Start Splunk
sudo /opt/splunk/bin/splunk start
```

## Verification
CLI
```bash
sudo -u splunk /opt/splunk/bin/splunk search "index=sysmon | stats count" -auth admin:splunkadmin123!
sudo -u splunk /opt/splunk/bin/splunk search "index=security | stats count" -auth admin:splunkadmin123!
```
Web UI
```bash
index=sysmon | stats count by sourcetype
index=security | stats count by sourcetype
```

## Common Errors & Fixes
Index Exists but No Data
```bash
sudo -u splunk /opt/splunk/bin/splunk fsck repair --all-buckets-one-index --index-name=sysmon
```

## fsck Permission Errors
```bash
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/sysmon/
```
Searches Slow or Incomplete
Fix:
Stop Splunk → run fsck → start Splunk

Why fsck Is Required (Splunk Internals)??
Splunk does not auto-discover copied index data.

It relies on:
- Bucket metadata
- Time range registration
- Journal state
  
When index folders are copied:
- Raw data exists 
- Splunk metadata does not 

fsck repair:
- Scans all buckets
- Rebuilds metadata
- Re-registers time ranges
- Makes data searchable again

Without fsck:
- Logs silently disappear
- DFIR timelines break
- Attacks become invisible

## Simplified Flow TLDR
1. Install Splunk
2. Create indexes
3. Stop Splunk
4. Copy index folders
5. Fix ownership
6. Run fsck
7. Start Splunk
8. Verify data

