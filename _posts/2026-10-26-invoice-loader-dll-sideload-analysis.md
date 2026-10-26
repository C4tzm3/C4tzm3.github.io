---
layout: post
title: "Signed by NetEase, Loaded by an Invoice: Analyzing a DLL Side-Loading Chain with an RC4-Protected Loader"
date: 2026-10-26
categories: [others]
tags: [malware-analysis, dll-side-loading, shellcode, rc4, threat-intelligence, dfir, windows, loader, crane]
description: "A static analysis of a fake invoice delivery (Invoice_20261026_Settlement_Report.zip) abusing a legitimately signed NetEase crash-handler binary for DLL side-loading. Covers an encrypted shellcode loader hidden in a rogue PE section, RC4 decryption with a hardcoded key, a staged payload download, and MITRE ATT&CK mapping."
---

A static analysis of a malware sample delivered as a fake invoice archive (`Invoice_20261026_Settlement_Report.zip`). The chain abuses a legitimately code-signed NetEase crash-handler executable for DLL side-loading, a trojanized `CrashHunter_PC3.dll` containing a custom in-memory loader hidden in a non-standard PE section, and an RC4-encrypted stage-2 bootstrap that retrieves the final payload over HTTPS.

![Extracting the sample](/assets/invoice2026/01_step1.png)

---

## Executive Summary

The sample arrives as a ZIP archive named to look like a financial settlement report, containing two files:

| File | Type | Size | Role |
|---|---|---|---|
| `2026102600987837.exe` | PE32+ GUI (x64) | 862 KB | Signed dropper/side-load host |
| `CrashHunter_PC3.dll` | PE32+ DLL (x64) | 3.3 MB | Trojanized loader (payload host) |

Key findings:

- The EXE is **signed with a valid NetEase (Hangzhou) Network Co., Ltd certificate** (DigiCert-issued, timestamped 2026-08-25). The binary itself is a genuine NetEase crash-reporting component (`bugreport.exe`) that references `dump.config` and loads `CrashHunter_PC3.dll` via `LoadLibrary`.
- The DLL masquerades as NetEase's crash-handler SDK (`NtUniSDK`, `MyCrashHandler`, `AppDumpWorker` classes) but contains a **rogue `.ljgyu` PE section** holding position-independent shellcode.
- The shellcode is a **custom reflective-style loader**: it resolves `KERNEL32` by walking the PEB, parses a PE embedded after a `CA FE BA BE` marker in the EXE, **RC4-decrypts it with the hardcoded key `AtkExEmbedKey2026`**, and executes it in memory.
- The decrypted stage (30,368 bytes) is a **download-and-execute bootstrap** containing an XOR-obfuscated (single-byte key `0x5A`) staging URL: `https://bmafhahl.eu.cc/files/awhfos.eu.cc.bin`
- The loader validates an `SCURLTPL` config template, downloads the `.bin` file, decrypts it with the same RC4 key, and invokes its entry point.

---

## Infection Chain

```
Invoice_20261026_Settlement_Report.zip
        │
        ▼
2026102600987837.exe  (signed NetEase bugreport.exe — victim double-clicks)
        │  LoadLibrary("CrashHunter_PC3.dll")   ← DLL side-loading
        ▼
CrashHunter_PC3.dll  (.ljgyu section — shellcode loader)
        │  1. Walk PEB → resolve KERNEL32 + API set (7 hashed functions)
        │  2. Read own EXE from disk
        │  3. Locate CA FE BA BE marker
        │  4. RC4-decrypt blob after marker (key: "AtkExEmbedKey2026")
        │  5. Execute stage-2 in memory
        ▼
Stage-2 bootstrap (30,368 bytes, SHA256: 3a0ead09...5482b)
        │  XOR 0x5A → reveals staging URL
        ▼
https://bmafhahl.eu.cc/files/awhfos.eu.cc.bin   ← final payload download
```

---

## Analysis

### 1. The Signed Side-Load Host

The EXE carries a valid Authenticode signature:

```
Subject : C=CN, O=NetEase (Hangzhou) Network Co., Ltd
Issuer  : DigiCert Trusted G4 Code Signing RSA4096 SHA384 2021 CA1
Timestamp: 2026-08-25
```

Internally, the binary identifies itself as NetEase's crash-reporting tool:

![File identification and hashes](/assets/invoice2026/02_step2.png)

- Manifest assembly name: `Game.app`
- References to `bugreport.exe`, `dump.config`, `CrashHunter_PC3.dll`
- Error string: `Could not load TheWorld_x64r.dll!`

![Side-loading strings in the EXE](/assets/invoice2026/03_step3.png)

The malicious logic relies on **search-order side-loading**: the signed EXE calls `LoadLibraryW("CrashHunter_PC3.dll")` at startup. Because the DLL is placed in the same directory, the trojanized copy wins over any legitimate version. PE compile timestamp of the EXE: **2026-08-11**.

### 2. The Trojanized CrashHunter_PC3.dll

The DLL impersonates NetEase's `NtUniSDK` crash-handler library.strings include:

- Classes: `MyCrashHandler`, `AppDumpWorker`, `GlobalTransId@NtUniSDK`
- Dump artifacts: `UE_FATAL_%llu_%llu_crash.zip`, `SNAPSHOT_%llu_%llu_crash.zip`, `BACKTRACE_SNAPSHOT_*`
- Legit-looking C2 camouflage: `https://appdump.nie.netease.com/upload` and `https://appdump.nie.easebar.com/upload`

Notable PE characteristics:

- Contains `.detourc` / `.detourd` sections — compiled with **Microsoft Detours** (API hooking framework)
- Non-standard section `.ljgyu` — 2,560 bytes, contains the malicious loader
- No exported names (exports by ordinal only)
- Signed with the same NetEase certificate; compile timestamp: **2025-04-11**

![Section table revealing the rogue .ljgyu section](/assets/invoice2026/05_step5.png)

The camouflage C2 strings baked into the DLL (designed to look like a benign crash-report uploader):

![Camouflage upload endpoints in the DLL](/assets/invoice2026/04_step4.png)

### 3. The .ljgyu Loader — PEB Walking and API Hashing

The shellcode implements a classic position-independent loader:

**Step 1 — PEB walk** (`gs:[0x60]` → `InMemoryOrderModuleList`) to locate `KERNEL32`:

```asm
mov rax, qword ptr gs:[0x60]      ; PEB
mov rdi, qword ptr [rax + 0x18]   ; LoaderData
add rdi, 0x20                     ; InMemoryOrderModuleList
mov rbx, qword ptr [rdi]          ; first entry (walks to find kernel32)
```

![Disassembly of the .ljgyu loader — PEB walk and RC4 key assembly](/assets/invoice2026/06_step6.png)

**Step 2 — API resolution via 32-bit hashes**, stored in a config struct validated by magic `0x7F`:

| Hash | Purpose (inferred) |
|---|---|
| `0x7C0017A5` | CreateFileW |
| `0x10FA6516` | ReadFile |
| `0x76DA08AC` | CloseHandle |
| `0x0FFD97FB` | GetFileSize / SetFilePointer |
| `0x91AFCA54` | VirtualAlloc |
| `0x7946C61B` | VirtualFree |
| `0x45B06D76` | CallWindowProc / entry invoker |

**Step 3 — RC4 implementation** is embedded directly (256-byte S-box KSA + PRGA loop) with the hardcoded key:

```
AtkExEmbedKey2026
```

The key is assembled on the stack as four DWORDs + a trailing byte: `AtkE`, `xEmb`, `edKe`, `y202`, `6`.

### 4. Payload Discovery — CA FE BA BE

The loader reads **its host EXE from disk** (`CreateFileW` with `GENERIC_READ`, `OPEN_EXISTING`), validates the MZ/PE header, then scans for the marker:

```asm
cmp byte ptr [r8 + rbp], 0xCA
cmp byte ptr [rax + rbp], 0xFE
cmp byte ptr [rax + rbp], 0xBA
cmp byte ptr [rax + rbp], 0xBE
```

In the sample, the `CA FE BA BE` marker sits at file offset `0xCB400`, immediately followed by an **RC4-encrypted blob**. Decrypting it with `AtkExEmbedKey2026` yields a 30,368-byte position-independent shellcode.

```
Payload SHA256: 3a0ead090043b333ee5718376a8215c2ef8863168b3a2d0a0460b6d0bab5482b
Payload MD5:    6fd872f3336a7f03b86002cdfe4ff7db
```

![RC4 decryption of the embedded payload and C2 extraction](/assets/invoice2026/07_step7.png)

### 5. Stage-2 — SCURLTPL Download-and-Execute Bootstrap

The decrypted stage contains a config block tagged `SCURLTPL` and an obfuscated string XOR-encoded with single-byte key `0x5A`:

```
Encoded: 322e2e2a2960757538373b3c323b3236743f2f743939753c33363f29753b2d323c3529743f2f74393974383334
Decoded: https://bmafhahl.eu.cc/files/awhfos.eu.cc.bin
```

Behavior of stage-2:

1. Allocates a 16-byte config structure and initializes it
2. Validates the `SCURLTPL` template (rejects if mismatched)
3. Downloads the `.bin` file from the staging URL over HTTPS (embedded HTTP client)
4. Decrypts the downloaded blob with the same RC4 key (`AtkExEmbedKey2026`)
5. Allocates RWX memory and transfers control to the final payload

The `.bin` naming convention and staged download pattern is consistent with loader frameworks that keep the final implant off-disk to evade AV/EDR file scanning.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Phishing: Malicious File | T1566.001 |
| Execution | User Execution: Malicious File | T1204.002 |
| Defense Evasion | DLL Side-Loading | T1574.002 |
| Defense Evasion | Obfuscated Files or Information (XOR, RC4) | T1027 |
| Defense Evasion | Software Packing / Encrypted Payload | T1027.002 |
| Defense Evasion | Process Injection: Portable Execution Injection | T1055.003 (in-memory execution) |
| Discovery | System Information Discovery (PEB/module walk) | T1082 |
| Command and Control | Application Layer Protocol: HTTPS | T1071.001 |
| Command and Control | Encrypted Channel | T1573 |
| Execution | Native API | T1106 |

---

## Indicators of Compromise (IOCs)

### File Hashes

| Artifact | SHA256 |
|---|---|
| ZIP archive | `9747df2f349e9d3b1ebc5aceb6e6ad3b495de0df6d90f1c548332eca2f9f7fdd` |
| `2026102600987837.exe` | `ba5a8e19125ef6f8b3ac52b1f2a8758cda206fd76ea73d401706659c67c4b531` |
| `CrashHunter_PC3.dll` | `0cc5e6f9fc0d7844c174d9b25ed84633e2afacaa2ef95b6e2f456062985caa8e` |
| Decrypted stage-2 shellcode | `3a0ead090043b333ee5718376a8215c2ef8863168b3a2d0a0460b6d0bab5482b` |

### Network

| Type | Indicator |
|---|---|
| URL | `https://bmafhahl.eu.cc/files/awhfos.eu.cc.bin` (staging — stage-2 download) |
| Domain | `bmafhahl.eu.cc` |
| Domain | `appdump.nie.easebar.com` (camouflage upload endpoint) |
| Domain | `appdump.nie.netease.com` (camouflage upload endpoint) |

### Host-Based

- Rogue PE section name: `.ljgyu` (in DLL)
- RC4 key string: `AtkExEmbedKey2026` (assembled on stack)
- Payload config marker: `SCURLTPL`
- Embedded blob marker: `CA FE BA BE` at file offset `0xCB400` in the EXE
- Crash-dump naming pattern in `%TEMP%`: `*_crash.zip` (e.g. `UE_FATAL_*_crash.zip`, `BACKTRACE_SNAPSHOT_*_crash.zip`)

---

## Detection Ideas

**YARA (conceptual):**

```yara
rule InvoiceLoader_CrashHunter_Sideload
{
    strings:
        $key = { 41 74 6b 45 78 45 6d 62 65 64 4b 65 79 32 30 32 }  // "AtkExEmbedKey2026" split across movs
        $cfg = "SCURLTPL"
        $sec = ".ljgyu"
        $n   = { CA FE BA BE }
    condition:
        uint16(0) == 0x5A4D and 2 of them
}
```

**Hunting queries:**

- `bugreport.exe` or any signed NetEase binary loading a `CrashHunter_PC3.dll` from a user-writable directory (Desktop, Downloads, Temp)
- TLS connections to freshly registered `.eu.cc` subdomains
- Executables in Downloads containing `CA FE BA BE` followed by >4 KB of high-entropy data

---

## Attribution

While the sample abuses a Chinese-signed binary (NetEase), several factors point away from a Chinese-nexus APT and toward a **commodity loader ecosystem**:

1. **Prior reporting of the same DLL family**: In July 2025, MalwareHunterTeam flagged a cluster of DLLs with the same overlay/Download-folder pattern — including `CrashHunter_PC.dll` and `AtkExEmbedKey2026.dll` — indicating this sample (`CrashHunter_PC3.dll`) is a newer, versioned iteration of an active campaign.

2. **Masquerade theme**: The `AtkExEmbedKey2026` RC4 key name mimics ASUS's legitimate ATK Package utility (`AtkExKeys`), consistent with loader families that abuse utility-software naming (some vendors track this builder as **CraneLoader**-style tooling).

3. **Infrastructure**: The staging domain `bmafhahl.eu.cc` uses a free dynamic-subdomain service with random naming — typical criminal infrastructure rather than the dedicated, long-lived C2 preferred by APT groups.

4. **Role in the chain**: The recovered stages are purely download-and-execute bootstrap logic — a **loader-for-hire** pattern that delivers a final payload (RAT/stealer) sold or operated elsewhere.

The closest publicly named family is **Aspidiex** (named by Zscaler ThreatLabz, 2024), attributed to **TA866** — a financially motivated actor known for invoice-themed malspam, DLL-based loaders, staged `.bin` downloads, and follow-on implants such as AHKBackdoor.

| Confidence | Attribution |
|---|---|
| Moderate | TA866 / TA567-adjacent commodity loader cluster (Aspidiex / CraneLoader ecosystem) |
| Possible | Loader-as-a-Service sold to multiple affiliates |
| Low | Chinese-nexus APT (NetEase abuse is likely convenience, not attribution signal) |

A notable builder detail: the hardcoded key contains **"2026"** — likely a year-based version marker — supporting the assessment of an actively maintained criminal loader family rather than a one-off operation.

---

## Conclusion

This sample demonstrates the continued abuse of **validly signed legitimate software** as a side-loading anchor, combined with a modular in-memory loader that never writes the final payload to disk. The use of a single hardcoded RC4 key (`AtkExEmbedKey2026`) across stages made static decryption straightforward once the `CA FE BA BE` marker was located — a reminder that even custom crypters often rely on reusable keys, enabling full payload recovery without any dynamic detonation.

---

*Sample analyzed in a static-only environment on macOS. No detonation was performed; behavioral details are inferred from disassembly of the loader and stage-2 bootstrap.*
