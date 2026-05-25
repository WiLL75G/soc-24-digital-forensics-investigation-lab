# Day 24 – SOC Tier 1 Incident Report: Digital Forensics Investigation Lab

---

## Incident Summary

- **Incident Type:** Full Endpoint Compromise DFIR Investigation
- **Severity:** Critical Credential Theft + Data Exfiltration Confirmed
- **Detection Method:** SIEM Alert → DFIR Investigation → Attack Timeline Reconstruction
- **Tools Referenced:** Splunk, Sysmon, Windows Event Logs, Prefetch, Registry Analysis, EDR
- **Status:** Complete Full Attack Chain Reconstructed, 47MB Exfiltration Confirmed

---

## Executive Summary

A full Digital Forensics and Incident Response (DFIR) investigation was conducted on WORKSTATION-22 (10.0.0.55) belonging to user mike.chen@nexuscorp.com. The investigation confirmed a complete attack chain from initial phishing email through to data exfiltration. A malicious Word macro delivered a PowerShell payload that downloaded a RAT, established persistence via registry and scheduled task, dumped LSASS credentials, moved laterally to FILE-SERVER-01, and exfiltrated 47MB of finance data to C2 server 45.131.214.85. The full attack chain was reconstructed with forensic evidence at every stage.

---

## Affected System

- **Hostname:** WORKSTATION-22
- **Internal IP:** 10.0.0.55
- **User:** mike.chen@nexuscorp.com
- **OS:** Windows 11 Pro
- **Secondary Host Affected:** FILE-SERVER-01 (10.0.0.10)
- **C2 Server:** 45.131.214.85
- **Data Exfiltrated:** 47MB Finance Q2 Reports

---

## Investigation Methodology

---

### 1. Evidence Collection Volatile Artefacts First

- Applied the forensics golden rule: Collect → Preserve → Analyse → Report
- Collected volatile evidence first running processes, network connections, logged in users
- Identified suspicious process: update.exe running from AppData\Roaming
- Identified outbound connection to 45.131.214.85:443 from update.exe
- All evidence hashed with SHA256 before and after collection
- Chain of custody documented — no original evidence modified

#### SOC Observations:

- Volatile evidence is lost on reboot always collect it first
- A process running from %APPDATA% is an immediate red flag legitimate software never runs from there
- SHA256 hashing of all evidence proves integrity and is required for any legal proceedings

---

### 2. Evidence Collection Non-Volatile Artefacts

- Exported Windows Event Logs Security, System, Application
- Analysed Prefetch files confirmed WINWORD.EXE spawning PowerShell
- Analysed Registry run keys found malicious persistence entry "WindowsUpdate"
- Identified recently modified files archive.zip created in temp directory
- Reviewed scheduled tasks found malicious task AUSessionConnect2

#### Key Event IDs Collected:
- Event 4688 — PowerShell spawned by WINWORD.EXE
- Event 4698 — Scheduled task created
- Event 4656 — LSASS handle requested (credential dumping)
- Event 4624 — Lateral movement logon to FILE-SERVER-01
- Event 4663 — Finance files accessed on file server

#### SOC Observations:

- Prefetch files prove execution even if the malware was deleted after running
- Registry run keys are the most common persistence mechanism always check them
- Event ID 4656 with LSASS as the target object is a confirmed credential dumping indicator

---

### 3. Attack Timeline Reconstruction

- Reconstructed full attack chain from T+0 (phishing email) to T+211min (containment)
- Total dwell time before detection: 3 hours 9 minutes
- 7 distinct attack phases identified and evidenced

#### Attack Chain:
```
08:14 UTC — Phishing email received
08:47 UTC — Malicious macro executed
08:48 UTC — PowerShell downloads update.exe
08:49 UTC — Persistence via registry run key
08:50 UTC — Scheduled task created
09:15 UTC — LSASS credential dump
09:45 UTC — Lateral movement to FILE-SERVER-01
10:02 UTC — 47 finance files accessed
10:30 UTC — Data staged in temp directory
10:47 UTC — 47MB exfiltrated to 45.131.214.85
11:15 UTC — SIEM alert fires
11:23 UTC — Host isolated
11:45 UTC — IR team activated
```

#### SOC Observations:

- The attacker had 3 hours of dwell time before detection earlier detection is the goal
- Every stage of the attack left forensic evidence DFIR is about finding and connecting the dots
- Timeline reconstruction is the deliverable that Tier 2 and IR teams need to act

---

## IOC Table

| Type | Value | Verdict |
|---|---|---|
| Affected Host | WORKSTATION-22 (10.0.0.55) | ❌ Compromised |
| Affected User | mike.chen@nexuscorp.com | ❌ Credentials stolen |
| Malicious File | Q2_Review.docm | ❌ Phishing attachment |
| Dropped Payload | update.exe | ❌ RAT dropper |
| Payload Path | C:\Users\mike.chen\AppData\Roaming\update.exe | ❌ Malicious |
| C2 IP | 45.131.214.85 | ❌ Confirmed C2 |
| Registry Key | HKCU\...\Run\WindowsUpdate | ❌ Persistence |
| Scheduled Task | AUSessionConnect2 | ❌ Persistence |
| Exfil Archive | archive.zip (47MB) | ❌ Data stolen |
| Secondary Host | FILE-SERVER-01 (10.0.0.10) | ⚠️ Accessed |

---

## MITRE ATT&CK Mapping

| Technique ID | Technique | Evidence |
|---|---|---|
| T1566.001 | Phishing: Spearphishing Attachment | Email logs |
| T1204.002 | User Execution: Malicious File | Prefetch, Event 4688 |
| T1059.001 | PowerShell | Event 4688, Script Block Log |
| T1105 | Ingress Tool Transfer | Proxy logs |
| T1547.001 | Registry Run Keys | Registry hive |
| T1053.005 | Scheduled Task | Event 4698 |
| T1003.001 | LSASS Memory | Event 4656, EDR alert |
| T1078 | Valid Accounts | Event 4624, 4648 |
| T1021.002 | SMB/Windows Admin Shares | Firewall, Event 4624 |
| T1005 | Data from Local System | File access logs |
| T1560.001 | Archive Collected Data | Sysmon Event 11 |
| T1041 | Exfiltration Over C2 Channel | Proxy, firewall logs |

---

## SOC Analyst Findings

- Full attack chain confirmed phishing to exfiltration in 3 hours 9 minutes
- Malicious Word macro delivered PowerShell payload via user execution
- update.exe established persistence via registry run key and scheduled task
- LSASS credential dump confirmed local admin credentials harvested
- Lateral movement to FILE-SERVER-01 using harvested credentials
- 47 finance files staged and exfiltrated to C2 server 45.131.214.85
- Total dwell time: 3 hours 9 minutes before SIEM detection

---

## SOC Analyst Response

- WORKSTATION-22 isolated via EDR immediately upon detection
- C2 IP 45.131.214.85 blocked at perimeter firewall
- mike.chen account disabled and password reset forced
- FILE-SERVER-01 access reviewed no further compromise found
- Malicious registry key and scheduled task removed
- update.exe quarantined and submitted for malware analysis
- All forensic evidence preserved with SHA256 hashes
- Incident escalated to Tier 2 and IR team with full timeline

---

## Analyst Insight

Digital forensics is detective work with timestamps. Every attacker action leaves a trace the job of the DFIR analyst is to find those traces, sequence them into a timeline, and present a story that answers four questions: How did they get in? What did they do? What did they take? How do we stop it happening again? In this investigation, the 3-hour dwell time before detection is the most important finding not because the analyst failed, but because it shows exactly where the detection gap is and how to close it.

---

## Learning Outcome

- Apply the DFIR golden rule Collect, Preserve, Analyse, Report
- Identify and collect volatile evidence before non-volatile
- Use Windows Event IDs to reconstruct attack activity
- Analyse Prefetch files, Registry keys, and Scheduled Tasks for forensic evidence
- Reconstruct a full attack timeline from forensic artefacts
- Map DFIR findings to 12 MITRE ATT&CK techniques
- Produce a forensic evidence report with chain of custody documentation

---

## Repository Structure

```
soc-24-digital-forensics-investigation-lab/
├── README.md
├── evidence/
├── reports/
│   └── evidence_collection.md
└── timeline/
    └── attack_timeline.md
```

---

## Conclusion

This lab demonstrates a complete Digital Forensics and Incident Response workflow. A full attack chain was reconstructed from forensic evidence phishing delivery, PowerShell execution, persistence, credential theft, lateral movement, and data exfiltration. Twelve MITRE ATT&CK techniques were mapped with supporting evidence. This project proves the ability to investigate a compromised endpoint, reconstruct what happened, and deliver the forensic evidence package that Tier 2 and IR teams need to respond effectively.
