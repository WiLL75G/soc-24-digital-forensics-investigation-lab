# Digital Forensics Evidence Collection Report

## Case Details

| Field | Details |
|---|---|
| **Case ID** | DFIR-2026-001 |
| **Date** | May 2026 |
| **Analyst** | James |
| **Incident Type** | Suspected Endpoint Compromise |
| **Affected Host** | WORKSTATION-22 (10.0.0.55) |
| **OS** | Windows 11 Pro |
| **Status** | Evidence Collected — Analysis In Progress |

---

## Chain of Custody

```
Evidence collected by: James (SOC Analyst Tier 1)
Collection method: Live forensics — system running
Collection time: 2026-05-23 09:00 UTC
Evidence stored: Encrypted container — AES-256
Hash verified: Yes — SHA256 checksums recorded
Tamper evidence: Digital signatures applied
```

---

## The Golden Rule of Digital Forensics

```
COLLECT → PRESERVE → ANALYSE → REPORT

Never modify evidence.
Always work on a copy.
Document every action taken.
Time-stamp everything.
```

---

## Phase 1 — Volatile Evidence (Collect First — Lost on Reboot)

Volatile evidence exists only in memory and active system state.
It is lost the moment the machine is powered off.
This must be collected FIRST.

### 1.1 Running Processes

**Command:**
```bash
# Windows
Get-Process | Select-Object Name, Id, Path, CPU | Sort-Object CPU -Descending

# Linux
ps aux --sort=-%cpu | head -20
```

**What to look for:**
```
- Processes running from temp directories (/tmp, %APPDATA%, %TEMP%)
- Processes with no file path listed (injected code)
- Legitimate process names with unusual parent processes
- High CPU processes with no clear purpose
- PowerShell or cmd.exe spawned by Office applications
```

**SOC Observation:**
Any process running from a temp folder is a confirmed red flag.
Legitimate software never runs from %TEMP%.

---

### 1.2 Network Connections

**Command:**
```bash
# Windows
netstat -ano | findstr ESTABLISHED

# Linux
ss -tulnp
netstat -an | grep ESTABLISHED
```

**What to look for:**
```
- Outbound connections to unknown external IPs
- Connections on unusual ports (not 80, 443, 53)
- Multiple connections to the same external IP
- Connections from processes that should not have network access
- Connections to Tor exit nodes or known C2 ranges
```

**SOC Observation:**
Cross-reference every external IP with VirusTotal and AbuseIPDB.
A process making outbound connections on port 4444 is almost always malware.

---

### 1.3 Logged In Users

**Command:**
```bash
# Windows
query user

# Linux
who
w
last | head -20
```

**What to look for:**
```
- Unknown or unexpected users currently logged in
- Multiple concurrent sessions for the same user
- Sessions at unusual times (3 AM logins)
- Users logged in from unexpected IP addresses
```

---

### 1.4 Open Files and Handles

**Command:**
```bash
# Windows (Sysinternals handle.exe required)
handle.exe -a

# Linux
lsof | grep -v "^COMMAND" | head -30
```

**What to look for:**
```
- Encrypted or locked files (ransomware indicator)
- Unusual files open in temp directories
- Database files open by unknown processes
```

---

## Phase 2 — Non-Volatile Evidence (Persists After Reboot)

### 2.1 Windows Event Logs

**Critical Event IDs for Forensics:**

```
4624  — Successful logon
4625  — Failed logon attempt
4648  — Logon with explicit credentials
4672  — Special privileges assigned (admin logon)
4688  — Process creation (requires audit policy)
4698  — Scheduled task created
4720  — User account created
4728  — User added to security group
4732  — User added to local administrators group
7045  — New service installed
```

**Command to export logs:**
```powershell
# Export Security log to CSV
Get-WinEvent -LogName Security -MaxEvents 1000 |
Select-Object TimeCreated, Id, Message |
Export-Csv -Path C:\forensics\security_events.csv
```

---

### 2.2 Prefetch Files

**What they are:**
Windows stores prefetch files for every application run.
They prove what programs were executed — even if deleted.

**Location:**
```
C:\Windows\Prefetch\
```

**What to look for:**
```
- Malware executables (random names like svc32.exe, update.exe)
- Tools like mimikatz.exe, psexec.exe, nc.exe (netcat)
- PowerShell with encoded commands
- Programs run from temp directories
```

---

### 2.3 Registry Keys

**Persistence locations — attackers love these:**
```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKLM\SYSTEM\CurrentControlSet\Services
```

**Command:**
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

**What to look for:**
```
- Unknown entries with random names
- Entries pointing to temp directories
- Entries that run PowerShell or cmd with encoded commands
- Recently added entries (correlate with incident timeline)
```

---

### 2.4 Recently Modified Files

**Command:**
```bash
# Windows — files modified in last 24 hours
Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue |
Where-Object {$_.LastWriteTime -gt (Get-Date).AddHours(-24)} |
Select-Object FullName, LastWriteTime | Sort-Object LastWriteTime -Descending

# Linux
find / -mtime -1 -type f 2>/dev/null | grep -v proc
```

---

## Phase 3 — File Hashing for Integrity

Every piece of evidence must be hashed to prove it has not been tampered with.

**Command:**
```bash
# Generate SHA256 hash — Windows
Get-FileHash -Algorithm SHA256 -Path C:\evidence\memory.dmp

# Generate SHA256 hash — Linux
sha256sum /evidence/memory.dmp

# Generate MD5 hash — Linux
md5sum /evidence/memory.dmp
```

**Evidence Hash Log:**
```
File: memory.dmp
MD5:    [record hash here]
SHA256: [record hash here]
Collected: 2026-05-23 09:00 UTC
Analyst: James
```

---

## Evidence Integrity Statement

All evidence collected in this investigation has been:

```
✅ Collected using forensically sound methods
✅ Hashed with SHA256 before and after collection
✅ Stored in an encrypted container
✅ Chain of custody documented
✅ No original evidence has been modified
✅ All analysis performed on verified copies
```
