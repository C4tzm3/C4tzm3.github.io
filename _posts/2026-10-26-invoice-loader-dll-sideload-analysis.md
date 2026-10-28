---
layout: post
title: "Hiding Malware Inside a Digital Signature: Analyzing a NetEase DLL Side-Loading Loader Delivered via WhatsApp Invoice"
date: 2026-10-26
categories: [others]
tags: [malware-analysis, dll-side-loading, shellcode, rc4, threat-intelligence, dfir, windows, loader, whatsapp]
description: "Full static analysis of a WhatsApp-delivered fake invoice (Invoice_20261026_Settlement_Report.zip): a signed NetEase EXE abused for DLL side-loading, a trojanized CrashHunter_PC3.dll with a shellcode loader hidden in the certificate table, RC4 decryption with a hardcoded key, and retrieval plus decryption of the stage-3 payload, including its custom ARX block cipher and socket-based backdoor. Includes IOCs and MITRE ATT&CK mapping."
---

A full static analysis of a malware sample delivered over **WhatsApp** as a fake invoice archive. The message asks the victim to "share the attached Statement of Account with your Finance Manager" and to "open the attachment using a computer", a social-engineering nudge that guarantees the ZIP is extracted and opened on a Windows workstation rather than a phone.

The chain abuses a legitimately code-signed NetEase crash-handler executable for DLL side-loading, a trojanized `CrashHunter_PC3.dll` whose entry point is patched to run a hidden shellcode loader stored inside the Authenticode certificate table, and an RC4-encrypted stage-2 bootstrap that retrieves the final payload over HTTPS.

**In plain language, for readers who want the short version:**

1. Someone sent a fake "invoice" over WhatsApp. Opening it on a Windows PC silently installs nothing visible: the malware hides inside files that look completely legitimate, one of which even carries a valid digital signature from a real company (NetEase, a large Chinese gaming company).
2. The malware's trick: it hides its own code in the part of a signed file that Windows does not check when verifying the signature. The signature still shows as valid.
3. It then downloads the final payload from the internet and runs it entirely in memory, so antivirus finds no malware file on disk.
4. The final payload is a full backdoor: an attacker who sent this "invoice" can potentially see the screen, steal files, and control the machine remotely.
5. If you received a similar WhatsApp "invoice", do not open it on a Windows PC. If you already did, check the IOCs section below.

![Extracting the sample](/assets/invoice2026/01_step1.png)

---

## The Lure

![WhatsApp lure message](/assets/invoice2026/whatsapp_lure.jpg)

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
- Stage 2 carries an `SCURLTPL` config block with an XOR-obfuscated (single-byte key `0x5A`) staging URL: `hxxps://bmafhahl[.]eu[.]cc/files/awhfos[.]eu[.]cc[.]bin`. It downloads the final payload and executes it entirely in memory.
- Stage 3 was **retrieved live from the staging server** and decrypted statically. It is a 119,938-byte position-independent loader wrapped around a partially encrypted x64 PE implant with socket-based C2, anti-debug checks, and COM usage. The container is **not** RC4-encrypted: it uses a custom 16-round ARX block cipher with a 16-byte counter key.

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

The shellcode implements a classic position-independent loader with no imports and no readable strings. Two techniques do the heavy lifting, and both are standard malware tradecraft:

- **PEB walking**: instead of calling `GetModuleHandle` (which antivirus tools and EDRs monitor), the shellcode reads the Windows Process Environment Block (PEB), the operating system's own in-memory list of loaded modules, directly, to find where kernel32.dll lives in memory without asking the operating system.
- **API hashing (ROR13)**: instead of storing function names like `CreateFileA` as readable text (which a strings scan would reveal), the shellcode stores only a numeric hash of each name. At runtime it walks kernel32's export table, hashes every function name the same way, and compares against the stored value. Same result, but nothing suspicious appears in the file.

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

### 6. Stage 3: Retrieving and Decrypting the Final Payload

Since the analysis is static-only, the staging server was contacted manually with `curl` (read-only retrieval, nothing executed). Public DNS-over-HTTPS resolvers gave the origin IP `103.212.97.252`, and a direct fetch returned the file.

![Retrieving awhfos.eu.cc.bin from the staging origin](/assets/invoice2026/08_stage3_retrieval.png)

```
File        : awhfos.eu.cc.bin
Size        : 119,938 bytes
SHA256      : 312896a21b25f8b5f1eee71131e9933dfdaba566bb8e20f2166a62121adf6175
MD5         : 9a9dcad691de57ba22af0ecc9ceca513
Entropy     : 7.96 bits/byte (encrypted container)
```

Two infrastructure details are worth noting. First, the host presents a **self-signed certificate** under issuer `Gateway CA - Cloudflare Managed G2` with a validity window of only **8 days** (issued 2026-10-26, the same day the sample was built). Short-lived TLS certificates are typical for malware staging infrastructure. Second, the file is served over HTTPS from a free `eu.cc` subdomain.

**Structure of the downloaded blob.** The file is not a PE. It is a position-independent loader blob:

| Offset | Size | Content |
|---|---|---|
| 0x0 | 5 | `CALL +0x1A266` stub (`E8 66 A2 01 00`) |
| 0x5 | 106,918 | Encrypted payload container (entropy 7.95) |
| 0x1A26B | ~13,020 | Plaintext x64 runtime: struct parsing, PEB-walking API resolver, command-line splitter |

The `CALL` at offset 0 transfers control to the runtime at `0x1A26B`, and the `pop rcx` that follows hands it a pointer to the encrypted container. The runtime treats the container as a structure: offset `+0x234` selects the decrypt path, `+0x238` gates execution, `+0x240` holds a semicolon-separated command line, and the first dword is the container size.

**The cipher is not RC4.** Stage 2 uses the same RC4 key as stage 1 (`AtkExEmbedKey2026`), so that was the first decryption attempt. It produced garbage. Disassembling the runtime revealed a custom block cipher at `+0x1D04E`:

- A 16-byte key stored inside the container acts as a **block counter**: after every 16-byte block the key is incremented by one (big-endian, with carry), forcing a fresh keystream per block
- Each block is expanded through a **16-round ARX permutation** over four 32-bit state words. ARX stands for Addition, Rotation, XOR: the three cheap operations the cipher repeats in loops, the same building blocks used by well-known ciphers like ChaCha20
- The state is also folded with a second 16-byte blob from the container header before being XORed into the ciphertext

Reimplementing the 40 lines of arithmetic in Python decrypts the full 106,918-byte container:

![Decrypting the stage-3 container with the custom ARX cipher](/assets/invoice2026/09_stage3_decrypt.png)

```
Decrypted container SHA256 : b273b96cfe72050acbb48b578591b57d1625d4f50ad43010dea64f29418e26ca
Carved x64 PE SHA256       : ef3b5575279d6e7f75b311e5b6864246ac65caf584d1edc387da813fc3fa7aec
PE compile timestamp       : 2026-10-12
```

**What is inside.** The decrypted container starts with a semicolon-separated list of libraries the implant loads dynamically:

```
ole32;oleaut32;wininet;mscoree;shell32
```

The embedded x64 PE (sections `.text`, `.rdata`, `.rsrc`; ImageBase `0x3D0000`) is an MSVC-linked implant. The loader keeps parts of its headers, sections, and string table encrypted at rest and reconstructs them in memory, so some strings are cut off mid-word in the on-disk form (`IsDebugge` really refers to `IsDebuggerPresent`, but the tail bytes only exist after runtime decryption). The strings below are reproduced exactly as they appear in the decrypted container:

| Category | Recovered strings (verbatim) |
|---|---|
| C2 networking | `WS2_32.dll`, `inet_ntop`, `WSAIoctl`, `StringToAddressA`, `Listen`, `Attach SOCKET`, `Send D` (truncated), `SSL en` (truncated), `CNNID: %Iu` |
| COM | `CoInitialize`, `SetProxy`, `Blanket` (`CoSetProxyBlanket`), `OLEAUT` |
| Anti-analysis | `IsDebugge` (truncated, `IsDebuggerPresent`), `RtlVirtualUnwinH` (truncated, `RtlVirtualUnwind`) |
| System | `numNetwork` (truncated), `WaitForM` (truncated, `WaitForMultipleObjects`), `DispatchM(` (truncated), `QueryPer` (truncated), `HeapAlloc` |
| Registry | `Microsoft\Cry` + `graphy` (split by encryption, `Software\Microsoft\Cryptography`), `LSOFTW` + `ARE\RRCfg` (split, `SOFTWARE\...RRCfg`) |
| Encoding | Full base64 alphabets (standard and custom) |

The truncation itself is a deliberate property of the build: because parts of the string table stay encrypted until execution, a static `strings` pass only reveals fragments, which complicates signature-based detection.

The presence of `mscoree` in the load list and `__clr`/`managed v` runtime fragments indicates .NET interaction capability. Combined with the socket-based command channel (`CNNID:` command identifiers), the implant is consistent with a **full backdoor rather than a simple downloader**: it enumerates network interfaces, maintains an SSL-protected command connection, and supports in-memory execution paths (`RtlVirtualUnwind`, thread management).

One additional artefact deserves clarification: `C:\Users\Public\Documents\1.txt`. This file does not exist on the staging server, so there is nothing to download; it is **written by stage 2 on the victim's machine** at runtime. Stage 2 contains a leftover debug routine that hex-dumps the downloaded stage-3 blob to this path before executing it. In an infected host, `1.txt` would simply contain a hexadecimal dump of `awhfos.eu.cc.bin` (the file retrieved above), which makes it a high-value forensic artefact: its content matches the staging server's file byte for byte.

### 7. What Happens After the Victim Runs It

Putting the whole post-execution sequence in order:

1. **Process start**: the victim double-clicks the signed EXE. No window or prompt indicates anything unusual; the process is a trusted NetEase crash reporter.
2. **DLL load**: the EXE loads `CrashHunter_PC3.dll` from its own folder. The patched entry point runs the real `DllMain` first, so the crash-handler SDK initializes normally and the process remains stable.
3. **Loader thread**: the `.ljgyu` stub resolves seven APIs by hashing, reads the EXE from disk, and pulls the 30 KB encrypted blob out of the certificate table. No file other than the EXE itself is touched.
4. **Stage 2 in memory**: the RC4-decrypted bootstrap decodes the staging URL, downloads `awhfos.eu.cc.bin` over HTTPS, allocates RWX memory, and calls into the blob's `CALL` stub. A leftover debug routine hex-dumps the download to `C:\Users\Public\Documents\1.txt`, which is a useful host artefact for IR.
5. **Stage 3 in memory**: the blob's runtime decrypts its own container with the custom ARX cipher, loads `ole32`, `oleaut32`, `wininet`, `mscoree`, and `shell32`, and runs the embedded backdoor entirely from memory. The implant never touches disk as a PE file.

### 8. Other Malicious Files Found

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
| Defense Evasion | Obfuscated Files or Information (RC4, XOR 0x5A, cert-table payload, custom ARX cipher) | T1027 |
| Defense Evasion | Dynamic API Resolution (ROR13 hashing) | T1027.007 |
| Discovery | Debugger Evasion (IsDebuggerPresent in the implant) | T1622 |
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
| Downloaded stage-3 blob (`awhfos.eu.cc.bin`) | `312896a21b25f8b5f1eee71131e9933dfdaba566bb8e20f2166a62121adf6175` |
| Decrypted stage-3 container | `b273b96cfe72050acbb48b578591b57d1625d4f50ad43010dea64f29418e26ca` |
| Carved stage-3 x64 PE implant | `ef3b5575279d6e7f75b311e5b6864246ac65caf584d1edc387da813fc3fa7aec` |

### Network

| Type | Indicator |
|---|---|
| URL | `hxxps://bmafhahl[.]eu[.]cc/files/awhfos[.]eu[.]cc[.]bin` (staging) |
| Domain | `bmafhahl[.]eu[.]cc` |
| IP | `103.212.97.252` (staging origin host) |
| TLS | Self-signed certificate, issuer `Gateway CA - Cloudflare Managed G2`, SHA256 fingerprint `DD:EE:AF:FD:3A:12:53:A1:33:B5:40:87:22:FE:27:5E:B8:44:9F:D9:BA:21:5B:92:BC:22:20:CD:01:FB:37:07`, 8-day validity |
| Domain | `appdump.nie.easebar.com` (camouflage upload endpoint) |
| Domain | `appdump.nie.netease.com` (camouflage upload endpoint) |

### Host-Based

- Rogue PE section name: `.ljgyu` in the DLL
- RC4 key string: `AtkExEmbedKey2026` (stack-assembled)
- Payload config marker: `SCURLTPL`; blob marker `CA FE BA BE` inside the certificate table
- Stage-3 container load list: `ole32;oleaut32;wininet;mscoree;shell32` (semicolon-separated, passed as a command line)
- Stage-3 implant artifacts: `CNNID:` command identifier prefix, mutex/event style strings under `Local\EVHS_`, UAC bypass string `Elevation:Administrator!new:{3E5FC7F9-9A51-4367-9063-A120244FBEC}` (CMSTPLUA COM elevation in stage 2)
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

## Recommended Actions

For defenders and IT teams, in order of priority:

1. **Block and flag the network indicators** listed in the IOCs: the domain `bmafhahl[.]eu[.]cc`, the IP `103.212.97.252`, and the URL pattern. Monitor DNS logs for any host that resolved the domain.
2. **Search endpoints for `C:\Users\Public\Documents\1.txt`**. If it exists on a workstation, that machine almost certainly ran this loader, and the file content is a hex dump that can be matched against the stage-3 hash in the IOC table.
3. **Alert on signed binaries with oversized certificate tables** (over ~24 KB). This costs almost nothing to deploy and defeats the hiding trick in this campaign and any copycat.
4. **Warn finance and admin staff** about invoice-themed files sent over WhatsApp or personal messaging apps. The lure here targeted a business workflow, and the "open using a computer" instruction is the tell: real invoicing portals do not need users to download and double-click anything.
5. **Do not rely on signature validation alone**: a file showing a valid NetEase signature does not mean it is clean, as demonstrated above.

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

This sample demonstrates the continued abuse of **validly signed legitimate software** as a side-loading anchor, combined with a modular in-memory loader that never writes the final payload to disk. The certificate-table hiding trick is effective against naive scanning but trivially detectable by size. The reuse of the hardcoded RC4 key (`AtkExEmbedKey2026`) made stage-2 recovery straightforward once the `CA FE BA BE` marker was located, while the final implant required disassembling its custom ARX block cipher, a reminder that a determined analyst can recover even freshly rolled crypto without any dynamic detonation. The final payload turns out to be a full socket-based backdoor rather than yet another downloader, which raises the practical impact of a single user opening a WhatsApp "invoice" on a workstation.

---

*Sample analyzed in a static-only environment on macOS. Nothing was executed. The staging server was contacted once with curl for a read-only retrieval of the stage-3 blob, which was then decrypted and dissected statically; no payload was ever run.*
