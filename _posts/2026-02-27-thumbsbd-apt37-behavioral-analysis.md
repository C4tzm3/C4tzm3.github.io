---
layout: post
title: "Behind THUMBSBD: Behavioral Analysis of an Advanced Backdoor Linked to APT37"
date: 2026-02-27
categories: [others]
tags: [apt37, malware-analysis, backdoor, north-korea, dfir, threat-intelligence, snakedropper, windows, shellcode]
description: "A behavioral analysis of THUMBSBD, a Windows backdoor linked to North Korean threat group APT37. Covers delivery via SNAKEDROPPER, C2 communication, in-memory shellcode execution, USB exfiltration, and MITRE ATT&CK mapping."
---

A behavioral analysis of THUMBSBD, a Windows backdoor associated with the North Korean threat group APT37, delivered via a shellcode dropper known as SNAKEDROPPER. This report covers how the malware operates after injection — from initialization to exfiltration — and how it attempts to evade detection.

---

## Executive Summary

THUMBSBD is a Windows backdoor deployed during the later stage of an attack chain through a component known as **SNAKEDROPPER**. Once active, THUMBSBD is responsible for maintaining persistent access to the compromised system.

After injection, the malware:
- Gathers system information including configuration, user identity, and network details
- Communicates with remote C2 servers over HTTPS to receive attacker instructions
- Executes commands, collects files, and stages data for exfiltration
- Interacts with removable media (USB) for offline/air-gapped exfiltration
- Uses multiple evasion techniques: process injection, XOR encryption, execution delays, and timestamp manipulation

---

## Threat Actor — APT37

| Attribute | Detail |
|---|---|
| **Name** | APT37 (also referenced as APT36 / Transparent Tribe in some intel) |
| **Aliases** | ScarCruft, Group123, Venus121, Inky Squid, Moldy Pisces, TA-RedAnt, Pearl Sleet, Red Eyes, ITG10 |
| **Origin** | North Korea |
| **Motivation** | State-Sponsored |
| **Targeted Sectors** | Government, Military, Financial Services, Manufacturing, Logistics, NGO |
| **Targeted Regions** | South Korea, United States, Vietnam, Hong Kong, Russian Federation |

---

## THUMBSBD Delivery

THUMBSBD does not exist as a standalone file on disk. It is embedded inside **SNAKEDROPPER**, a raw x86 shellcode dropper, and delivered directly into process memory.

**SNAKEDROPPER** characteristics:
- **Size**: 10 MB
- Contains the full **Ruby 3.3.0 runtime** used for persistence
- THUMBSBD payload is embedded at offset `0x964F20`, XOR-encoded with key `0x0A`
- **No import table** — all Windows API calls are resolved at runtime by hash, making static analysis significantly harder

**Injection flow:**
1. SNAKEDROPPER selects a legitimate Windows system process as the injection target
2. Decodes THUMBSBD in place using the XOR key
3. Injects into the target process using `VirtualAllocEx`, `WriteProcessMemory`, and `CreateRemoteThread`
4. THUMBSBD is now active inside a legitimate process — no file written to disk at any stage

---

## Behaviour Analysis

### Initialization and Working Directory Setup

The first action THUMBSBD performs after injection is setting up its working environment. It locates `%LOCALAPPDATA%` and creates **seven subdirectories** under a folder named `TnGtp`. These directories are used to store encrypted data, stage files for exfiltration, and receive operator commands.

![Figure 1 - Initialization Function (0x00401340)](/assets/images/others/thumbsbd-apt37/fig1-initialization.png)

On startup, THUMBSBD:
- Resolves `%LOCALAPPDATA%`
- Collects victim identity (username, hostname, OS version, 32/64-bit)
- Builds seven TnGtp working paths and creates the directories
- Writes the encrypted victim profile to `TN.dat`

---

### Victim Identification

THUMBSBD assigns each victim a permanent hardware identifier called **LUUID**, derived from the motherboard UUID via WMI. This ID does not change even after Windows is reinstalled, allowing the operator to permanently track the same physical machine.

![Figure 2 - Hardware UUID Query (0x00408470)](/assets/images/others/thumbsbd-apt37/fig2-hardware-uuid.png)

The malware queries `Win32_ComputerSystemProduct` to retrieve the hardware UUID. An MD5 hash of hardware data is also computed using the Windows CryptoAPI. Unlike usernames or IP addresses, the hardware UUID is stable and uniquely identifies the physical machine.

---

### Encrypted Victim Profile — TN.dat

All collected victim data is encrypted before being saved to `TN.dat`:
- Encryption uses **XOR key `0x83`**, applied as a 32-bit dword (`0x83838383`), processing 32 bytes at a time
- The file is deleted and recreated fresh on each run
- Without the key, `TN.dat` cannot be read by an analyst or forensic tool

![Figure 3 - Encrypted Profile Write (0x00401600)](/assets/images/others/thumbsbd-apt37/fig3-encrypted-profile.png)

---

### Reconnaissance Command Execution

THUMBSBD silently collects system information by running standard Windows commands via `cmd.exe`:

| Function (Address) | Command | Purpose |
|---|---|---|
| `ReconCmd_Ipconfig (0x407240)` | `/c ipconfig /all` | Full network adapter config, IP addresses, DNS |
| `ReconCmd_Netstat (0x4073d0)` | `/c netstat -ano` | Active connections, listening ports, process PIDs |
| `ExecuteReconCommand (0x402940)` | `/c dxdiag /t` + arbitrary | Hardware enumeration and operator-issued commands |
| `ReconCmd_Ping (0x407560)` | `/c ping 8.8.8.8` + `ping google.com` | Connectivity test and latency |

![Figure 4 - Recon Command Execution (0x00402940)](/assets/images/others/thumbsbd-apt37/fig4-recon-commands.png)

Each recon command follows the same pattern: runs via `cmd.exe` with no visible window (`nShow=0`), output is redirected to a randomly-named temp file, the malware waits for completion, collects the output, then deletes the temp file.

---

### C2 Communication

THUMBSBD communicates with **three hardcoded C2 servers** using the standard Windows internet library (WinInet). All requests use a **Chrome 121 browser User-Agent** string to blend with normal web traffic.

| C2 Domain | Endpoint |
|---|---|
| `https://www.philion.store` | `/star/main.php` |
| `https://www.homeatedke.store` | `/star/main.php` |
| `https://www.hightkdhe.store` | `/star/main.php` |

```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.3112.113 Safari/537.36
```

The malware monitors a local `TnGtp` folder for files dropped by the C2. When a new file arrives it is validated and dispatched.

---

### C2 Payload Validation and Dispatch

Every file sent by the C2 operator carries an **encrypted header**. THUMBSBD decrypts it and checks two security values to confirm the file is a legitimate operator command. Any file that fails this check is deleted immediately.

![Figure 5 - Payload Header Validation (0x00404640)](/assets/images/others/thumbsbd-apt37/fig5-payload-validation.png)

Valid files are routed based on a type code:
- **Type 1** — runs a command
- **Type 0** — stores a data file
- **Type 2** — executes a program only if the victim's hardware ID matches (LUUID-targeted delivery)

---

### Full Operator Command Surface

A separate function handles **seven distinct operator command types**:

| Type | Capability | Detail |
|---|---|---|
| 1 | Drive Enumeration | Lists all connected drives — type, label, total and free space |
| 2 | File Exfiltration | Reads a filename from the payload and sends that specific file to C2 |
| 3 | Shell Command | Reads a command string and executes it via `cmd.exe` |
| 4 | Shellcode Delivery | Receives a shellcode blob and runs it entirely in memory |
| 5 | Remote File Delete | Receives a filename and deletes that file on the victim machine |
| 6 | USB Scan Trigger | Instructs THUMBSBD to scan connected USB drives immediately |
| 7 | Victim Profile Update | Receives 4780 bytes and rewrites the encrypted `TN.dat` profile |

![Figure 6 - Full Command Dispatcher (0x004023e0)](/assets/images/others/thumbsbd-apt37/fig6-command-dispatcher.png)

**Type 4** is the most significant — it allows delivery and execution of arbitrary malware in memory at any time. Types 5 and 7 give the operator file deletion and remote profile update capabilities.

---

### In-Memory Shellcode Execution

When a Type 4 command arrives, THUMBSBD loads and runs the payload entirely in memory:

![Figure 7 - In-Memory Shellcode Execution (0x00404d20)](/assets/images/others/thumbsbd-apt37/fig7-shellcode-exec.png)

- `VirtualAlloc` with `PAGE_EXECUTE_READWRITE (0x40)` creates a region that can both store and execute code
- The shellcode is copied in before being executed via a **direct function pointer call**
- **Two `Sleep()` calls** are inserted before copy and before execution to defeat time-limited sandboxes
- Memory is freed with `VirtualFree` after execution — no file ever touches disk

---

### Command Deduplication — del.dat

THUMBSBD keeps an encrypted log of every command already executed in `del.dat`. Before running any new command, it checks this log:
- If the command is already present — it is **skipped**
- If new — it is **appended in encrypted form** before execution proceeds

This prevents the same command from running twice even if the C2 re-sends it.

![Figure 8 - del.dat Command Log](/assets/images/others/thumbsbd-apt37/fig8-del-dat.png)

---

### USB Staging — Offline Exfiltration Channel

THUMBSBD copies collected data to USB drives using a folder named **`$RECYCLE .BIN`** — almost identical to the real Windows Recycle Bin folder but with a **space before BIN**. All staged files are marked **HIDDEN** so they do not appear in normal folder views.

![Figure 9 - USB Staging (0x00402f60)](/assets/images/others/thumbsbd-apt37/fig9-usb-staging.png)

Combined with `ScanUSBDrive_StageFiles`, this creates a **complete bidirectional offline C2 channel** — the operator can reach victims without internet access, including in restricted or air-gapped environments.

---

### Timestamp Manipulation

THUMBSBD hides its activity from forensic investigators by resetting file timestamps. When it modifies a file it:

1. Reads the original **creation, access, and write timestamps**
2. Deletes and recreates the file (resetting Windows NTFS metadata)
3. Restores the original timestamps using `SetFileTime`

![Figure 10 - Timestamp Manipulation (0x004040d0)](/assets/images/others/thumbsbd-apt37/fig10-timestamp.png)

This technique can **bypass forensic tools** that rely on NTFS timestamps to reconstruct investigation timelines.

---

## MITRE ATT&CK Mapping

| Tactic | Technique ID | Description |
|---|---|---|
| Execution | T1059.003 | Executes recon commands through Windows command shell (ipconfig, netstat, ping) |
| Defense Evasion | T1027 | XOR encryption to obfuscate internal data and stored files |
| Defense Evasion | T1055.002 | Executes malicious code within the memory of another process |
| Defense Evasion | T1620 | Loads and executes shellcode directly from memory without writing to disk |
| Defense Evasion | T1036 | Uses folder name similar to Windows Recycle Bin (`$RECYCLE .BIN`) to hide staged files |
| Defense Evasion | T1497.003 | Uses execution delays to evade automated sandbox analysis |
| Defense Evasion | T1070.006 | Restores original file timestamps to hide evidence of file modification |
| Discovery | T1082 | Collects system information — OS version, host details |
| Discovery | T1016 | Retrieves network configuration and active connection information |
| Discovery | T1033 | Identifies current user and computer name |
| Discovery | T1120 | Enumerates connected removable devices (USB drives) |
| Collection | T1005 | Collects files from the local system |
| Collection | T1025 | Scans and collects files from removable media |
| Collection | T1074 | Stages collected data in local directories before exfiltration |
| Command and Control | T1071.001 | Communicates with remote servers using HTTPS web traffic |
| Command and Control | T1132 | Encodes and compresses data before transmission |
| Exfiltration | T1041 | Transfers collected data through the C2 channel |
| Exfiltration | T1052.001 | Uses removable media to stage data for offline exfiltration |

---

## Indicators of Compromise (IOC)

### File Hashes

| Hash | File |
|---|---|
| `e654df84fd6dc02ca1b312ff856ef2ca88b42a72bab31ea3168965cb946cf16e` | SNAKEDROPPER |
| `6a1242eae17593d6033ded8ee17a8eed863fa0347f251b2e40ffac89de97689b` | THUMBSBD |

### Domain / Network Indicators

| Indicator | Type |
|---|---|
| `www[.]philion.store` | THUMBSBD C2 |
| `www[.]homeatedke.store` | THUMBSBD C2 |
| `www[.]hightkdhe.store` | THUMBSBD C2 |
| `/star/main.php` | C2 endpoint (all three domains) |

### Host Indicators

| Indicator | Description |
|---|---|
| `%LOCALAPPDATA%\TnGtp\TN.dat` | Encrypted victim profile |
| `%LOCALAPPDATA%\TnGtp\...\del.dat` | Encrypted command log |
| `%LOCALAPPDATA%\TnGtp\...\exe.dat` | Payload staging file |
| `SOFTWARE\Microsoft\TnGtp` | Victim configuration registry key |
| XOR key `0x83` | THUMBSBD internal encryption key |
