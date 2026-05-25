# Attack Timeline Reconstruction

## Case ID: DFIR-2026-001
## Host: WORKSTATION-22 (10.0.0.55)
## Analyst: James

---

## Overview

This timeline reconstructs the full attack chain on WORKSTATION-22
based on forensic artefacts collected during the investigation.
All times are in UTC.

---

## Timeline

### Day 1 Initial Access

```
08:14 UTC T+0
Event: Phishing email received by user mike.chen@nexuscorp.com
Subject: "Q2 Performance Review Action Required"
Attachment: Q2_Review.docm (malicious Word macro)
Evidence: Exchange mail logs, email gateway alert

08:47 UTC T+33min
Event: User opens Q2_Review.docm in Microsoft Word
Evidence: Prefetch file WINWORD.EXE-[hash].pf
          Registry recent documents list

08:47 UTC T+33min
Event: Malicious macro executes
Parent process: WINWORD.EXE (PID 4821)
Child process: PowerShell.exe -EncodedCommand [base64]
Evidence: Windows Event ID 4688 process creation
          Sysmon Event ID 1 process creation with command line

08:48 UTC T+34min
Event: PowerShell downloads second stage payload
URL: http://45.131.214.85/update.exe
File saved: C:\Users\mike.chen\AppData\Roaming\update.exe
Evidence: Proxy logs, PowerShell script block logging
          Sysmon Event ID 11 file creation

08:48 UTC T+34min
Event: update.exe executed
File hash (SHA256): [record from investigation]
Evidence: Prefetch file UPDATE.EXE-[hash].pf
          Windows Event ID 4688
```

---

### Day 1 Persistence Established

```
08:49 UTC T+35min
Event: Attacker establishes persistence via registry run key
Key: HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
Value: "WindowsUpdate" = "C:\Users\mike.chen\AppData\Roaming\update.exe"
Evidence: Registry hive analysis
          Sysmon Event ID 13 registry value set

08:50 UTC T+36min
Event: Scheduled task created for persistence
Task name: \Microsoft\Windows\WindowsUpdate\AUSessionConnect2
Command: PowerShell.exe -WindowStyle Hidden -EncodedCommand [base64]
Evidence: Windows Event ID 4698 scheduled task created
          C:\Windows\System32\Tasks\ directory
```

---

### Day 1 Credential Harvesting

```
09:15 UTC T+61min
Event: Mimikatz-style credential dumping detected
Process: update.exe attempts to access LSASS memory
Evidence: Windows Event ID 4656 handle to LSASS requested
          Sysmon Event ID 10 process access to lsass.exe
          EDR alert credential dumping technique detected

09:17 UTC T+63min
Event: Local admin credentials harvested
Account: WORKSTATION-22\LocalAdmin
Method: LSASS memory dump
Evidence: EDR telemetry, memory artefacts
```

---

### Day 1 Lateral Movement

```
09:45 UTC — T+91min
Event: Attacker uses harvested credentials for lateral movement
Source: WORKSTATION-22 (10.0.0.55)
Destination: FILE-SERVER-01 (10.0.0.10)
Protocol: SMB port 445
Evidence: Windows Event ID 4624 logon type 3 (network)
          Windows Event ID 4648 explicit credential logon
          Firewall logs SMB traffic between workstations

10:02 UTC — T+108min
Event: Attacker accesses shared drive on FILE-SERVER-01
Path: \\FILE-SERVER-01\Finance\Q2_Reports\
Files accessed: 47 files viewed, 12 downloaded
Evidence: Windows Event ID 4663 object access
          File server access logs
```

---

### Day 1 Data Exfiltration

```
10:30 UTC T+136min
Event: Data staged for exfiltration
Location: C:\Users\mike.chen\AppData\Local\Temp\archive.zip
Contents: 12 finance files from FILE-SERVER-01
Size: 47MB
Evidence: File creation logs, Sysmon Event ID 11

10:47 UTC T+153min
Event: Data exfiltrated to C2 server
Destination: 45.131.214.85:443
Protocol: HTTPS (encrypted)
Data volume: 47MB outbound
Evidence: Proxy logs, firewall logs, NetFlow data
          SIEM alert large outbound transfer detected
```

---

### Day 1 Detection & Response

```
11:15 UTC T+181min
Event: SIEM alert fires large outbound data transfer
Rule: Outbound transfer > 10MB to external IP
Analyst: William — SOC Tier 1

11:23 UTC T+189min
Event: SOC investigation begins
Actions: Source IP 45.131.214.85 enriched confirmed malicious
         Host WORKSTATION-22 isolated via EDR

11:45 UTC T+211min
Event: Incident escalated to Tier 2 and IR team
Full attack chain identified
Evidence preservation initiated
```

---

## Full Attack Chain Summary

```
INITIAL ACCESS
Phishing email → Malicious macro → PowerShell download
T1566.001 → T1204.002 → T1059.001

EXECUTION
PowerShell encoded command → update.exe dropped and executed
T1059.001 → T1105

PERSISTENCE
Registry run key + Scheduled task
T1547.001 → T1053.005

CREDENTIAL ACCESS
LSASS memory dump → Local admin credentials harvested
T1003.001

LATERAL MOVEMENT
SMB with harvested credentials → File server accessed
T1021.002 → T1078

COLLECTION
Finance files staged in temp directory
T1005 → T1560.001

EXFILTRATION
47MB sent to C2 server over HTTPS
T1041 → T1048.002
```

---

## MITRE ATT&CK Full Mapping

| Technique ID | Technique | Evidence |
|---|---|---|
| T1566.001 | Phishing: Spearphishing Attachment | Email logs |
| T1204.002 | User Execution: Malicious File | Prefetch, Event 4688 |
| T1059.001 | PowerShell | Event 4688, Script Block Log |
| T1105 | Ingress Tool Transfer | Proxy logs |
| T1547.001 | Registry Run Keys | Registry hive |
| T1053.005 | Scheduled Task | Event 4698 |
| T1003.001 | LSASS Memory | Event 4656, EDR |
| T1078 | Valid Accounts | Event 4624, 4648 |
| T1021.002 | SMB/Windows Admin Shares | Firewall, Event 4624 |
| T1005 | Data from Local System | File access logs |
| T1560.001 | Archive Collected Data | Sysmon Event 11 |
| T1041 | Exfiltration Over C2 Channel | Proxy, firewall logs |
