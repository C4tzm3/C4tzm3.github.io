---
layout: post
title: "Hiding Malware Inside a Digital Signature: Analyzing a NetEase DLL Side-Loading Loader Delivered via WhatsApp Invoice"
date: 2026-10-26
categories: [others]
tags: [malware-analysis, dll-side-loading, shellcode, rc4, threat-intelligence, dfir, windows, loader, whatsapp]
description: "Full static analysis of a WhatsApp-delivered fake invoice (Invoice_20261026_Settlement_Report.zip): a signed NetEase EXE abused for DLL side-loading, a trojanized CrashHunter_PC3.dll with a shellcode loader hidden in the certificate table, RC4 decryption with a hardcoded key, and a staged payload download. Includes post-execution behaviour, IOCs, and MITRE ATT&CK mapping."
---

A full static analysis of a malware sample delivered over **WhatsApp** as a fake invoice archive. The message asks the victim to "share the attached Statement of Account with your Finance Manager" and to "open the attachment using a computer", a social-engineering nudge that guarantees the ZIP is extracted and opened on a Windows workstation rather than a phone.

The chain abuses a legitimately code-signed NetEase crash-handler executable for DLL side-loading, a trojanized `CrashHunter_PC3.dll` whose entry point is patched to run a hidden shellcode loader stored inside the Authenticode certificate table, and an RC4-encrypted stage-2 bootstrap that retrieves the final payload over HTTPS.

![Extracting the sample](/assets/invoice2026/01_step1.png)

---

## The Lure

![WhatsApp lure message](/assets/invoice2026/whatsapp_lure.png)

| Item | Value |
|---|---|
| Delivery | WhatsApp, forwarded ZIP attachment |
| Message | "Please share the attached Statement of Account with your Finance Manager for checking. For a complete view of the details, kindly open the attachment using a computer. Thank you." |
| Archive | `Invoice_20261026_Settlement_Report.zip` (2,038,927 bytes) |
| Contents | `2026102600987837.exe` (862,888 bytes) and `CrashHunter_PC3.dll` (3,376,576 bytes, hidden file attribute set) |

The lure is targeted at a business workflow rather than a person: the attachment name looks like a generated document ID, the message references a finance process, and the "open using a computer" instruction exists only because the payload is a Windows executable. Nothing in the archive is an invoice. The theme exists to make the victim open it quickly and without suspicion.

---

## Executive Summary

The sample arrives as a ZIP archive named to look like a financial settlement report. Nothing inside is an invoice; the theme pushes the victim to open it quickly. It contains two files:

| File | Type | Size | Role |
|---|---|---|---|
| `2026102600987837.exe` | PE32+ GUI (x64) | 862,888 bytes | Signed side-load host (legit NetEase `bugreport.exe`) |
| `CrashHunter_PC3.dll` | PE32+ DLL (x64) | 3,376,576 bytes | Trojanized loader (hidden file attribute set) |

Key findings:

- The EXE is **signed with a valid NetEase (Hangzhou) Network Co., Ltd certificate** (DigiCert-issued). The DLL carries the same certificate but its checksum field is zeroed and its signature no longer matches the file; Windows only surfaces the EXE signature to the user.
- The DLL entry point is patched: the original `DllMain` runs first, then execution flows into a rogue `.ljgyu` section containing position-independent shellcode.
- The shellcode is a **custom in-memory loader**: it resolves APIs by ROR13-add export hashing, reads the running EXE from disk, locates the `CA FE BA BE` marker inside the Security (certificate table) directory, and **RC4-decrypts a 30,368-byte stage-2 bootstrap with the hardcoded key `AtkExEmbedKey2026`**.
- The Authenticode certificate table is **excluded from the signature hash**, so the appended encrypted payload does not break the signature. Most scanners treat the cert table as signature data and skip it.
- Stage 2 carries an `SCURLTPL` config block with an XOR-obfuscated (single-byte key `0x5A`) staging URL: `hxxps://bmafhahl[.]eu[.]cc/files/awhfos[.]eu[.]cc[.]bin`. It downloads the final payload, decrypts it with the same RC4 key, and executes it entirely in memory.
- Stage 3 was **not retrieved** during this analysis (no detonation, no C2 contact), so the final implant family remains unconfirmed.

---

## Attack Flow

![Attack flow](/assets/invoice2026/attack_flow.png)

---

## Analysis

### 1. Why the EXE Looks Trustworthy

The EXE carries a valid Authenticode signature:

```
Subject : C=CN, O=NetEase (Hangzhou) Network Co., Ltd
Issuer  : DigiCert Trusted G4 Code Signing RSA4096 SHA384 2021 CA1
Timestamp: 2026-08-25
```

![File identification and hashes](/assets/invoice2026/02_step2.png)

Internally, the binary identifies itself as NetEase's crash-reporting tool:

- Manifest assembly name: `Game.app`
- References to `bugreport.exe`, `dump.config`, `CrashHunter_PC3.dll`
- Error string: `Could not load TheWorld_x64r.dll!`

![Side-loading strings in the EXE](/assets/invoice2026/03_step3.png)

The malicious logic relies on **search-order side-loading**: the signed EXE calls `LoadLibraryW("CrashHunter_PC3.dll")` at startup. Because the DLL sits in the same directory, the trojanized copy wins over any legitimate version. The victim sees a validly signed, trusted application. PE compile timestamp of the EXE: **2026-08-11**.

### 2. The Trojanized CrashHunter_PC3.dll

The DLL impersonates NetEase's `NtUniSDK` crash-handler library. Strings include:

- Classes: `MyCrashHandler`, `AppDumpWorker`, `GlobalTransId@NtUniSDK`
- Dump artifacts: `UE_FATAL_%llu_%llu_crash.zip`, `SNAPSHOT_%llu_%llu_crash.zip`, `BACKTRACE_SNAPSHOT_*`
- Legit-looking camouflage endpoints: `https://appdump.nie.netease.com/upload` and `https://appdump.nie.easebar.com/upload`

![Camouflage upload endpoints in the DLL](/assets/invoice2026/04_step4.png)

Notable PE characteristics:

- Contains `.detourc` / `.detourd` sections, indicating it was built with **Microsoft Detours** (an API hooking framework)
- Non-standard section `.ljgyu` (2,560 bytes, executable, entropy 5.75) containing the malicious loader
- Exports by ordinal only
- PE header `CheckSum = 0` while still carrying the NetEase signing certificate: the file was modified after signing
- DLL entry point `0x228D98` patched to call the original `DllMain` (so the SDK still initializes normally) and then the `.ljgyu` stub
- Compile timestamp: **2025-04-11**

![Section table revealing the rogue .ljgyu section](/assets/invoice2026/05_step5.png)

### 3. The .ljgyu Loader: PEB Walking and API Hashing

The shellcode implements a classic position-independent loader with no imports and no readable strings.

**Step 1: PEB walk** (`gs:[0x60]` -> `InMemoryOrderModuleList`) to locate `KERNEL32`:

```asm
mov rax, qword ptr gs:[0x60]      ; PEB
mov rdi, qword ptr [rax + 0x18]   ; LoaderData
add rdi, 0x20                     ; InMemoryOrderModuleList
mov rbx, qword ptr [rdi]          ; first entry (walks to find kernel32)
```

![Disassembly of the .ljgyu loader: PEB walk and RC4 key assembly](/assets/invoice2026/06_step6.png)

**Step 2: API resolution via 32-bit ROR13-add hashes**, filling a seven-entry function table:

| Hash | API |
|---|---|
| `0x7C0017A5` | CreateFileA |
| `0x10FA6516` | ReadFile |
| `0x76DA08AC` | SetFilePointer |
| `0x0FFD97FB` | CloseHandle |
| `0x91AFCA54` | VirtualAlloc |
| `0x7946C61B` | VirtualProtect |
| `0x45B06D76` | GetModuleFileNameA |

**Step 3: RC4 key assembly**. The key `AtkExEmbedKey2026` is built on the stack from five `mov` immediates (`AtkE`, `xEmb`, `edKe`, `y202`, `6`), so a plain `strings` scan finds nothing.

### 4. The Payload Hiding Trick: Certificate Table Abuse

This is the most interesting part of the chain. The loader:

1. Calls `GetModuleFileNameA(NULL)` to get the path of the running EXE
2. Opens and reads its own file from disk
3. Parses the PE header and reads the **Security directory (data directory index 4)**
4. Searches that blob for the `CA FE BA BE` marker
5. Copies everything after the marker, RC4-decrypts it, marks the memory executable with `VirtualProtect(PAGE_EXECUTE_READ)`, and jumps to it

In this sample the certificate table spans 41,128 bytes at file offset `0xC8A00`:

```
Cert table @ 0xC8A00, size 41,128 bytes
  [ 0 .. 10,744]     legit PKCS#7 signature (NetEase, DigiCert chain) - parses cleanly
  [10,744 .. 10,752] CA FE BA BE marker
  [10,752 .. 41,128] RC4-encrypted stage-2 (30,368 bytes)
```

The Authenticode specification excludes the certificate table from the signed hash, so appending data there does not invalidate the signature. The file on disk shows no high-entropy overlay and no suspicious sections; the payload is invisible to tools that only look after the last section. The tell is size: normal code-signing certificates are 6 to 12 KB, not 41 KB.

```
Payload SHA256: 3a0ead090043b333ee5718376a8215c2ef8863168b3a2d0a0460b6d0bab5482b
Payload MD5:    6fd872f3336a7f03b86002cdfe4ff7db
```

![RC4 decryption of the embedded payload and C2 extraction](/assets/invoice2026/07_step7.png)

### 5. Stage 2: SCURLTPL Download-and-Execute Bootstrap

The decrypted stage contains a config block tagged `SCURLTPL` and an obfuscated string XOR-encoded with single-byte key `0x5A`:

```
Encoded: 322e2e2a2960757538373b3c323b3236743f2f743939753c33363f29753b2d323c3529743f2f74393974383334
Decoded: https://bmafhahl.eu.cc/files/awhfos.eu.cc.bin
```

### 6. What Happens After the Victim Runs It

Putting the whole post-execution sequence in order:

1. **Process start**: the victim double-clicks the signed EXE. No window or prompt indicates anything unusual; the process is a trusted NetEase crash reporter.
2. **DLL load**: the EXE loads `CrashHunter_PC3.dll` from its own folder. The patched entry point runs the real `DllMain` first, so the crash-handler SDK initializes normally and the process remains stable.
3. **Loader thread**: the `.ljgyu` stub resolves seven APIs by hashing, reads the EXE from disk, and pulls the 30 KB encrypted blob out of the certificate table. No file other than the EXE itself is touched.
4. **Stage 2 in memory**: the RC4-decrypted bootstrap decodes the staging URL, downloads `awhfos.eu.cc.bin` over HTTPS, decrypts it with the same key, allocates RWX memory, and calls into it. A leftover debug routine hex-dumps the download to `C:\Users\Public\Documents\1.txt`, which is a useful host artefact for IR.
5. **Stage 3**: never written to disk as a file. Its family and objective (RAT, stealer, further loader) are unconfirmed because the C2 was not contacted during this static analysis.

### 7. Other Malicious Files Found

The archive contains exactly two files, both analysed:

- `2026102600987837.exe`: legitimate NetEase binary, **not modified** apart from its use as a side-load host. It is part of the malicious chain, but only in combination with the DLL.
- `CrashHunter_PC3.dll`: the malicious component. All malicious logic lives in the patched entry point plus the `.ljgyu` section; the rest of the DLL is genuine crash-handler SDK code, which explains why it passes casual inspection.

No additional droppers, batch files, or registry changes are present in the archive. Persistence was not observed in this sample: the loader runs at every execution but no Run keys, scheduled tasks, or service creation logic exist in the recovered stages. Redelivery through messaging apps appears to be the primary (re)infection vector.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Phishing: Malicious File | T1566.001 |
| Execution | User Execution: Malicious File | T1204.002 |
| Defense Evasion | DLL Side-Loading | T1574.002 |
| Defense Evasion | Code Signing: abuse of trusted signed binary | T1553.002 |
| Defense Evasion | Obfuscated Files or Information (RC4, XOR 0x5A, cert-table payload) | T1027 |
| Defense Evasion | Dynamic API Resolution (ROR13 hashing) | T1027.007 |
| Execution | In-memory shellcode execution | T1059 / T1620 |
| Command and Control | Ingress Tool Transfer (HTTPS download of stage 3) | T1105 |
| Command and Control | Encrypted Channel | T1573 |

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
| URL | `hxxps://bmafhahl[.]eu[.]cc/files/awhfos[.]eu[.]cc[.]bin` (staging) |
| Domain | `bmafhahl[.]eu[.]cc` |
| Domain | `appdump.nie.easebar.com` (camouflage upload endpoint) |
| Domain | `appdump.nie.netease.com` (camouflage upload endpoint) |

### Host-Based

- Rogue PE section name: `.ljgyu` in the DLL
- RC4 key string: `AtkExEmbedKey2026` (stack-assembled)
- Payload config marker: `SCURLTPL`; blob marker `CA FE BA BE` inside the certificate table
- Oversized certificate table: 41 KB vs the normal 6 to 12 KB
- Debug artefact path: `C:\Users\Public\Documents\1.txt`
- Filename pattern: `Invoice_<YYYYMMDD>_Settlement_Report.zip` with a 16-digit numeric EXE name

---

## Detection Ideas

**YARA (conceptual):**

```yara
rule InvoiceLoader_CrashHunter_Sideload
{
    strings:
        $key = { 41 74 6b 45 78 45 6d 62 65 64 4b 65 79 32 30 32 }
        $cfg = "SCURLTPL"
        $sec = ".ljgyu"
        $n   = { CA FE BA BE }
    condition:
        uint16(0) == 0x5A4D and 2 of them
}
```

**Hunting queries:**

- Any signed NetEase binary (original filename `bugreport.exe`) running from Downloads, Temp, or a user-extracted folder and loading `CrashHunter_PC3.dll` from the same path. Legit NetEase games run from their install directory.
- Signed PE whose Security directory exceeds ~24 KB or contains data after the PKCS#7 chain.
- Outbound HTTPS from a signed process to free `eu.cc` subdomains shortly after process start.

---

## Attribution

While the sample abuses a Chinese-signed binary (NetEase), several factors point away from a Chinese-nexus APT and toward a **commodity loader ecosystem**:

1. **Prior reporting of the same DLL family**: In July 2025, MalwareHunterTeam flagged a cluster of DLLs with the same overlay and Downloads-folder pattern, including `CrashHunter_PC.dll` and `AtkExEmbedKey2026.dll`, indicating this sample (`CrashHunter_PC3.dll`) is a newer, versioned iteration of an active campaign.
2. **Masquerade theme**: The `AtkExEmbedKey2026` RC4 key name mimics ASUS's legitimate ATK Package utility (`AtkExKeys`), consistent with loader families that abuse utility-software naming.
3. **Infrastructure**: The staging domain `bmafhahl.eu.cc` uses a free dynamic-subdomain service with random naming: typical criminal infrastructure rather than the dedicated, long-lived C2 preferred by APT groups.
4. **Role in the chain**: The recovered stages are purely download-and-execute bootstrap logic, a **loader-for-hire** pattern that delivers a final payload operated elsewhere.

The closest publicly named family is **Aspidiex** (named by Zscaler ThreatLabz, 2024), attributed to **TA866**, a financially motivated actor known for invoice-themed malspam, DLL-based loaders, staged `.bin` downloads, and follow-on implants such as AHKBackdoor.

| Confidence | Attribution |
|---|---|
| Moderate | TA866 / TA567-adjacent commodity loader cluster (Aspidiex / CraneLoader ecosystem) |
| Possible | Loader-as-a-Service sold to multiple affiliates |
| Low | Chinese-nexus APT (NetEase abuse is likely convenience, not an attribution signal) |

A notable builder detail: the hardcoded key contains **"2026"**, likely a year-based version marker, supporting the assessment of an actively maintained criminal loader family rather than a one-off operation.

---

## Conclusion

This sample demonstrates the continued abuse of **validly signed legitimate software** as a side-loading anchor, combined with a modular in-memory loader that never writes the final payload to disk. The certificate-table hiding trick is effective against naive scanning but trivially detectable by size. The use of a single hardcoded RC4 key (`AtkExEmbedKey2026`) across stages made static decryption straightforward once the `CA FE BA BE` marker was located, a reminder that even custom crypters often rely on reusable keys, enabling full payload recovery without any dynamic detonation.

---

*Sample analyzed in a static-only environment on macOS. Nothing was executed and the C2 was not contacted; post-execution details are inferred from disassembly of the loader and stage-2 bootstrap.*
