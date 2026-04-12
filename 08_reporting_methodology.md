# 08 — Reporting & Methodology
> eCPPTv3 Cheatsheet | Attacker-Mindset Reference

---

## Table of Contents
1. [Penetration Testing Methodology](#1-penetration-testing-methodology)
2. [PTES Phases Reference](#2-ptes-phases-reference)
3. [Note-Taking During the Exam](#3-note-taking-during-the-exam)
4. [Evidence Collection](#4-evidence-collection)
5. [Finding Structure & Write-Up](#5-finding-structure--write-up)
6. [CVSS Scoring](#6-cvss-scoring)
7. [Risk Ratings](#7-risk-ratings)
8. [Report Structure](#8-report-structure)
9. [Executive Summary Writing](#9-executive-summary-writing)
10. [Technical Finding Examples](#10-technical-finding-examples)
11. [eCPPTv3 Exam-Specific Notes](#11-ecpptv3-exam-specific-notes)
12. [Quick-Win Checklist](#12-quick-win-checklist)

---

## 1. Penetration Testing Methodology

```
Standard Pentest Flow:
─────────────────────────────────────────────────────────────────────
1. Pre-Engagement      → Scope, rules of engagement, legal sign-off
2. Reconnaissance      → Passive + active information gathering
3. Scanning & Enum     → Port scanning, service fingerprinting
4. Vulnerability ID    → Map services to known CVEs, misconfigs
5. Exploitation        → Gain initial access, validate vulns
6. Post-Exploitation   → PrivEsc, lateral movement, persistence
7. Reporting           → Document everything, write findings
8. Remediation Retest  → Verify fixes (not always in scope)
─────────────────────────────────────────────────────────────────────
```

```
Attack Chain / Kill Chain Thinking:
  Initial Access → Execution → Persistence → Privilege Escalation
  → Defense Evasion → Credential Access → Discovery
  → Lateral Movement → Collection → Exfiltration / Impact
```

---

## 2. PTES Phases Reference

> PTES = Penetration Testing Execution Standard — the framework most commonly
> referenced in professional reporting and eCPPT.

```
Phase 1: Pre-Engagement Interactions
  - Define scope (IP ranges, domains, excluded systems)
  - Rules of Engagement (RoE) — what is/isn't allowed
  - Emergency contacts and escalation paths
  - Legal authorisation — signed statement of work
  - Timing windows (business hours only? 24/7?)

Phase 2: Intelligence Gathering (Recon)
  - OSINT: WHOIS, DNS, LinkedIn, Shodan, Google dorks
  - Technical: subdomain enum, email harvesting
  - Deliverable: target profile, attack surface map

Phase 3: Threat Modelling
  - Identify high-value assets and likely threat actors
  - Map attack paths from external to crown jewels
  - Prioritise what to test based on business risk

Phase 4: Vulnerability Analysis
  - Automated scanning: Nessus, OpenVAS, Nikto
  - Manual testing: version checks, config review
  - Cross-reference NVD / ExploitDB / searchsploit

Phase 5: Exploitation
  - Validate vulnerabilities — prove exploitability
  - Gain initial foothold
  - Document exact steps, commands, timestamps

Phase 6: Post-Exploitation
  - Privilege escalation
  - Lateral movement across network
  - Data access / exfiltration simulation
  - Demonstrate business impact

Phase 7: Reporting
  - Executive summary for non-technical stakeholders
  - Technical findings with reproduction steps
  - Risk ratings and remediation guidance
  - Appendices: tool output, screenshots, payloads
```

---

## 3. Note-Taking During the Exam

> Good notes = good report. Take notes in real time — never rely on memory.

### 3.1 What to Record for Every Action

```
For every significant action during the exam, record:

  Timestamp   → when it happened (date + time)
  Target      → IP, hostname, service, port
  Action      → what you did (command or tool used)
  Output      → what the result was (copy raw output)
  Finding     → what vulnerability/weakness was demonstrated
  Impact      → what could an attacker do with this?
  Screenshot  → visual proof of command + output
```

### 3.2 Recommended Tools

```bash
# CherryTree — hierarchical note-taking, great for pentests
# Obsidian — markdown-based, links between notes
# Notion — collaborative, good for structured templates
# Joplin — open source, markdown, good search
# tmux + vim — terminal-native logging

# KeepNote (older but simple)
# Greenshot / Flameshot — screenshot with annotation
```

### 3.3 Directory Structure for the Exam

```bash
# Create at the start — keep everything organised
mkdir -p ~/exam/{scans,loot,exploits,screenshots,report}
mkdir -p ~/exam/scans/{nmap,web,smb,vuln}
mkdir -p ~/exam/loot/{hashes,creds,keys,files}
mkdir -p ~/exam/report/{findings,evidence}

# Save all tool output with timestamps
nmap -sV -sC $TARGET -oA ~/exam/scans/nmap/initial_$(date +%Y%m%d_%H%M)
```

### 3.4 tmux Logging

```bash
# Log entire terminal session automatically (great for exam evidence)
tmux new-session -s exam
# In tmux, enable logging:
# Ctrl+B then : → then type:
pipe-pane -o "cat >> ~/exam/session_log_$(date +%Y%m%d).txt"

# Script command — log everything printed to terminal
script -a ~/exam/terminal_$(date +%Y%m%d_%H%M).log
```

### 3.5 Screenshot Naming Convention

```
# Use consistent naming so you can find evidence quickly in the report:
YYYY-MM-DD_HH-MM_TARGET-IP_FINDING-NAME.png

# Examples:
2025-04-15_14-32_10.10.10.5_ms17010-confirmed.png
2025-04-15_15-10_10.10.10.5_root-shell-obtained.png
2025-04-15_16-45_172.16.1.10_ntlm-hash-dumped.png
```

---

## 4. Evidence Collection

> The rule: if you didn't screenshot it, it didn't happen.
> Capture evidence at every critical step — before and after exploitation.

### 4.1 What Screenshots to Take

```
For EVERY finding, capture:

  [ ] Vulnerability proof — the scanner output or manual confirmation
  [ ] Exploit execution — the command/payload you ran
  [ ] Shell/access obtained — terminal showing id, whoami, hostname
  [ ] Privilege level — show user context (root, SYSTEM, DA)
  [ ] Sensitive data accessed — files, hashes, config data (redact if needed)
  [ ] Network reach — ip a, ipconfig to show network position
  [ ] Timestamps visible in terminal or tool output
```

### 4.2 Linux Screenshot Command

```bash
# Flameshot — annotated screenshots (highlight, arrows, boxes)
flameshot gui

# scrot — quick terminal screenshot
scrot ~/exam/screenshots/$(date +%Y%m%d_%H%M%S).png

# Show date/time in terminal before screenshot
date && id && hostname
```

### 4.3 Capturing Command Output

```bash
# Pipe output to file AND show on screen simultaneously
id | tee ~/exam/loot/whoami_output.txt

# Save nmap output in all formats
nmap -sV -sC $TARGET -oA ~/exam/scans/nmap/target_full

# Save crackmapexec output
crackmapexec smb $TARGET -u admin -p pass 2>&1 | tee ~/exam/scans/cme_output.txt

# Save all impacket output
python3 secretsdump.py CORP/admin:pass@$TARGET 2>&1 | tee ~/exam/loot/secretsdump.txt
```

### 4.4 Proof Files (eCPPT Style)

```bash
# On every compromised host, capture a "proof" screenshot showing:
# - hostname
# - current user (root/SYSTEM or DA equivalent)
# - proof.txt if one exists (CTF-style exam)
# - IP address of the machine

hostname && id && ip a && cat /root/proof.txt
# Windows:
hostname && whoami && ipconfig && type C:\Users\Administrator\Desktop\proof.txt
```

---

## 5. Finding Structure & Write-Up

> Each vulnerability finding should be a self-contained document.
> A reviewer should be able to understand, reproduce, and fix the issue
> from your write-up alone — without asking you any questions.

### 5.1 Finding Template

```
─────────────────────────────────────────────────
FINDING: [Short descriptive title]
─────────────────────────────────────────────────

Title:       MS17-010 EternalBlue — Remote Code Execution
Severity:    Critical
CVSS Score:  9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
Host:        10.10.10.5 (WIN-DC01)
Service:     SMB — TCP 445
CVE:         CVE-2017-0144

─── Description ───────────────────────────────────
What the vulnerability is, why it exists, and what class
of weakness it represents. 2–4 sentences, technical but
readable by a senior developer.

─── Impact ────────────────────────────────────────
What an attacker can achieve by exploiting this.
Frame in business terms: "An unauthenticated attacker
can gain SYSTEM-level access and fully compromise the host,
including all data stored on it and credentials in memory."

─── Evidence ──────────────────────────────────────
Step-by-step reproduction:
  1. Command / action taken
  2. Tool output (paste raw output)
  3. Screenshot reference: [figure-01.png]
  4. Result achieved

─── Affected Systems ──────────────────────────────
  - 10.10.10.5 (WIN-DC01, Windows Server 2008 R2)
  - 10.10.10.6 (WIN-WS01, Windows 7 SP1)

─── Remediation ───────────────────────────────────
Specific, actionable steps to fix:
  1. Apply Microsoft patch MS17-010 immediately.
  2. Disable SMBv1 if not required: Set-SmbServerConfiguration -EnableSMB1Protocol $false
  3. Block inbound SMB (TCP 445) at the perimeter firewall.
  4. Implement network segmentation to limit lateral movement risk.

─── References ────────────────────────────────────
  - https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2017/ms17-010
  - https://nvd.nist.gov/vuln/detail/CVE-2017-0144
  - https://www.exploit-db.com/exploits/42315
─────────────────────────────────────────────────
```

### 5.2 Finding Title Conventions

```
# Good titles — specific, scannable, action-oriented:
"MS17-010 EternalBlue — Unauthenticated Remote Code Execution via SMB"
"Apache 2.4.49 Path Traversal and Remote Code Execution (CVE-2021-41773)"
"WordPress Admin Credentials Exposed via Default Login"
"Kerberoastable Service Account with Weak Password"
"SMB Null Session — Unauthenticated Domain User Enumeration"
"NTLM Hash Captured via LLMNR Poisoning"
"Unquoted Service Path Privilege Escalation"

# Bad titles — vague, unclear, unhelpful:
"SQL vulnerability"
"Weak password"
"Privilege escalation found"
"Security issue on server"
```

---

## 6. CVSS Scoring

> CVSS v3.1 — Common Vulnerability Scoring System.
> Used to standardise severity across findings.

### 6.1 Base Score Metrics

```
Metric               Options
──────────────────────────────────────────────────────────────
Attack Vector (AV)   Network(N) | Adjacent(A) | Local(L) | Physical(P)
Attack Complexity    Low(L) | High(H)
Privileges Required  None(N) | Low(L) | High(H)
User Interaction     None(N) | Required(R)
Scope                Unchanged(U) | Changed(C)
Confidentiality      None(N) | Low(L) | High(H)
Integrity            None(N) | Low(L) | High(H)
Availability         None(N) | Low(L) | High(H)
```

### 6.2 Score Ranges

```
Score Range    Severity
─────────────────────────
0.0            None
0.1 – 3.9      Low
4.0 – 6.9      Medium
7.0 – 8.9      High
9.0 – 10.0     Critical
```

### 6.3 Common CVSS Strings for eCPPT Findings

```
# Remote code execution — unauthenticated, network
AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H     → 9.8  Critical
# Example: EternalBlue, Log4Shell, ShellShock

# Remote code execution — authenticated
AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H     → 8.8  High
# Example: Authenticated SQLi → OS command exec

# Local privilege escalation
AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H     → 7.8  High
# Example: SUID abuse, sudo misconfiguration

# Information disclosure — network, no auth
AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N     → 7.5  High
# Example: SNMP public community, LDAP anon bind

# Weak credentials / default login
AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N     → 6.5  Medium
# Example: FTP anonymous, Tomcat default creds

# Information disclosure — requires auth
AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N     → 6.5  Medium

# Missing patches / configuration weakness
AV:N/AC:H/PR:N/UI:N/S:U/C:L/I:L/A:N     → 4.8  Medium
```

```bash
# Online CVSS calculator — use to generate exact scores
# https://www.first.org/cvss/calculator/3.1
```

---

## 7. Risk Ratings

> When CVSS alone isn't enough, apply contextual risk rating.
> Business impact + exploitability = real-world risk.

```
Risk     = Likelihood × Impact

Likelihood factors:
  - Is it remotely exploitable or local only?
  - Does it require credentials?
  - Is public exploit code available?
  - Is the service internet-facing?

Impact factors:
  - What data/systems are exposed?
  - What is the blast radius (one host vs whole domain)?
  - What is the business consequence (data loss, downtime, compliance)?
```

```
Simplified Risk Matrix:

               Impact
               Low       Medium    High
Likelihood ┌─────────┬──────────┬──────────┐
High       │ Medium  │  High    │ Critical │
Medium     │  Low    │  Medium  │  High    │
Low        │  Low    │  Low     │  Medium  │
           └─────────┴──────────┴──────────┘
```

---

## 8. Report Structure

> The report is your final deliverable — it demonstrates the full value of the test.

```
─────────────────────────────────────────
REPORT STRUCTURE (Professional Template)
─────────────────────────────────────────

1. Cover Page
   - Report title, client name, date, classification (CONFIDENTIAL)
   - Tester name(s), report version

2. Table of Contents

3. Document Control
   - Version history, distribution list

4. Executive Summary (1–2 pages)
   - Engagement overview (non-technical)
   - Overall security posture assessment
   - Key findings summary (count by severity)
   - Top 3–5 most critical issues in plain language
   - Strategic recommendations

5. Scope & Methodology
   - In-scope assets (IPs, domains, systems)
   - Out-of-scope items
   - Testing period / dates
   - Methodology used (PTES / OWASP / NIST)
   - Tools used

6. Attack Narrative (optional but valuable)
   - Story of the engagement from attacker's POV
   - Shows how findings chain together
   - "We started at X, found Y, which led to Z"

7. Findings (Technical)
   - One section per finding
   - Ordered by severity: Critical → High → Medium → Low → Info
   - Each finding: title, severity, CVSS, description,
     impact, evidence, reproduction steps, remediation

8. Appendices
   A. Scan Output (nmap, nikto, etc.)
   B. Tool Configuration
   C. Payload / PoC Code Used
   D. Full Hash/Credential List (marked confidential)
   E. Glossary (for non-technical readers)
─────────────────────────────────────────
```

---

## 9. Executive Summary Writing

> Written for: CISOs, managers, board members — non-technical decision-makers.
> Goal: Tell them what matters, what's at risk, and what to do next.
> No jargon. No technical commands. Business language only.

### 9.1 Structure

```
Paragraph 1 — Engagement Overview
  What was tested, when, and why. One or two sentences.

Paragraph 2 — Overall Posture
  How did the organisation perform overall?
  Use adjectives: strong, adequate, weak, critical risk.
  Avoid: "we found X vulnerabilities" without context.

Paragraph 3 — Key Findings
  3–5 bullets of the most impactful findings in plain English.
  Each bullet: what was found + what it means to the business.

Paragraph 4 — Positive Observations (optional)
  What was done well? Balanced view builds trust.

Paragraph 5 — Strategic Recommendations
  Top 3 priorities for the next 30/60/90 days.
  Frame as business priorities, not technical tasks.
```

### 9.2 Example Executive Summary Paragraph

```
During the assessment, [Company] demonstrated significant security
weaknesses across both its external and internal environments.
Testers were able to gain unauthenticated access to the internal
network through an unpatched vulnerability in the VPN appliance,
and subsequently obtained full administrative control of the
Active Directory domain within four hours of initial access.
This level of access would allow an attacker to exfiltrate all
company data, deploy ransomware across the entire environment,
or cause extended operational downtime.

Priority remediation should focus on: (1) immediate patching of
internet-facing systems, (2) enforcing multi-factor authentication
across all remote access services, and (3) reviewing administrative
account privileges to implement least-privilege access controls.
```

---

## 10. Technical Finding Examples

### Finding 1 — MS17-010 EternalBlue

```
Title:       MS17-010 EternalBlue — Unauthenticated Remote Code Execution
Severity:    Critical
CVSS:        9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
Host:        10.10.10.5
Service:     SMB — TCP 445

Description:
The target host is running an unpatched version of Windows Server 2008 R2
that is vulnerable to CVE-2017-0144, commonly known as EternalBlue. This
vulnerability exists in the SMBv1 protocol implementation and allows an
unauthenticated remote attacker to execute arbitrary code with SYSTEM-level
privileges by sending specially crafted packets to TCP port 445.

Impact:
An unauthenticated attacker with network access to TCP port 445 can achieve
complete compromise of this host, gaining SYSTEM-level access. All data
stored on the system, credentials cached in memory, and the system's role
within the domain (including any Domain Controller functionality) are fully
accessible to the attacker.

Evidence:
  1. Vulnerability confirmed via Nmap:
     nmap -p 445 --script=smb-vuln-ms17-010 10.10.10.5
     [Screenshot: figure-01-eternalblue-confirmed.png]

  2. Exploited using Metasploit module exploit/windows/smb/ms17_010_eternalblue:
     [Screenshot: figure-02-exploit-execution.png]

  3. SYSTEM shell obtained:
     meterpreter > getuid → Server username: NT AUTHORITY\SYSTEM
     [Screenshot: figure-03-system-shell.png]

Remediation:
  1. Apply Microsoft security update MS17-010 immediately.
  2. Disable SMBv1 protocol: Set-SmbServerConfiguration -EnableSMB1Protocol $false
  3. Block inbound TCP 445 at the network perimeter firewall.
  4. Segment the network to restrict SMB traffic to authorised hosts only.

References:
  - https://docs.microsoft.com/security-updates/SecurityBulletins/2017/ms17-010
  - https://nvd.nist.gov/vuln/detail/CVE-2017-0144
```

### Finding 2 — Kerberoasting

```
Title:       Weak Kerberoastable Service Account Password
Severity:    High
CVSS:        7.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N)
Host:        10.10.10.10 (corp.local Domain Controller)
Service:     Kerberos — TCP 88

Description:
The domain contains service accounts configured with Service Principal Names
(SPNs) and weak passwords. Any authenticated domain user can request a
Kerberos TGS ticket for these accounts, which is encrypted with the account's
NTLM hash. The ticket can be extracted and cracked offline without generating
any domain alerts.

Impact:
An attacker with any valid domain credential can obtain the NTLM hash of
privileged service accounts and crack it offline. If the service account has
elevated privileges (e.g., Domain Admin), this results in full domain
compromise. In this assessment, the svc_sql account (member of Domain Admins)
was cracked in under 60 seconds.

Evidence:
  1. Kerberoastable accounts identified:
     GetUserSPNs.py CORP/jdoe:Password1 -dc-ip 10.10.10.10 -request
     [Screenshot: figure-04-spn-list.png]

  2. TGS hash cracked offline with Hashcat:
     hashcat -m 13100 hash.txt rockyou.txt → svc_sql:Summer2023!
     [Screenshot: figure-05-hash-cracked.png]

  3. Domain Admin access obtained with cracked credentials:
     [Screenshot: figure-06-domain-admin-access.png]

Remediation:
  1. Set all service account passwords to random, 25+ character values
     using Group Managed Service Accounts (gMSA) where possible.
  2. Remove unnecessary SPN registrations.
  3. Remove service accounts from privileged groups (Domain Admins).
  4. Enable AES-only Kerberos encryption to make cracking significantly harder.
  5. Monitor for unusual TGS request volume (potential Kerberoasting activity).
```

---

## 11. eCPPTv3 Exam-Specific Notes

```
eCPPTv3 Exam Format:
  - Fully practical, hands-on lab environment
  - 14 days total: 7 days testing + 7 days reporting
  - Report submitted as PDF
  - Requires demonstrated compromise + documentation

Key things the report MUST show:
  [ ] How each host was initially compromised (exact steps)
  [ ] What vulnerability was exploited (CVE or class)
  [ ] Proof of access (screenshot with hostname + user + IP)
  [ ] Privilege level achieved (root/SYSTEM/DA)
  [ ] Lateral movement path (how you moved between hosts)
  [ ] Pivoting methodology (how you reached internal segments)
  [ ] Remediation recommendation for each finding

Screenshot requirements (include all of these):
  [ ] Nmap output confirming service/version
  [ ] Exploit execution command
  [ ] Shell obtained (id/whoami visible)
  [ ] Hostname and IP of compromised host
  [ ] Any proof.txt content if present
  [ ] Network path showing pivot route

Report tone and format:
  - Professional, third-person where appropriate
  - Findings ordered Critical → High → Medium → Low
  - Each finding independently reproducible from write-up
  - Executive summary written for non-technical reader
  - No slang, no overly casual language
```

### 11.1 Attack Chain Documentation Template

```
Document your attack path as a narrative chain:

  HOST 1 (10.10.10.5 — External)
  ├── Vuln: MS17-010 EternalBlue (CVE-2017-0144)
  ├── Access: SYSTEM shell via Metasploit
  ├── Loot: Local SAM hashes, found creds in config file
  └── Pivot: Discovered internal subnet 172.16.1.0/24 via ipconfig

      HOST 2 (172.16.1.10 — Internal Web Server)
      ├── Access: SMB Pass-the-Hash using NTLM from HOST 1
      ├── Vuln: Apache 2.4.49 path traversal + RCE
      ├── Access: www-data shell via CVE-2021-41773
      ├── PrivEsc: SUID binary abuse → root
      └── Loot: Found domain service account creds in /var/www/config.php

          HOST 3 (172.16.1.2 — Domain Controller)
          ├── Access: svc_backup domain account (from HOST 2 loot)
          ├── Vuln: svc_backup has DCSync rights (ACL misconfiguration)
          ├── Action: DCSync → dumped all domain NTLM hashes
          └── Impact: Full domain compromise — all accounts compromised
```

---

## 12. Quick-Win Checklist

```
DURING THE EXAM — NOTE-TAKING
[ ] Create directory structure at start: scans/loot/exploits/screenshots/report
[ ] Enable terminal logging (script or tmux pipe-pane)
[ ] Record timestamp, target, command, output for EVERY action
[ ] Screenshot: service/version confirmed → exploit run → shell obtained → proof
[ ] Save all raw tool output to files (nmap -oA, CME tee, etc.)
[ ] Name screenshots consistently: DATE_TIME_HOST_FINDING.png
[ ] For each host: capture hostname + id/whoami + ip a in same screenshot

DURING THE EXAM — ATTACK CHAIN
[ ] Document which host gave access to the next host
[ ] Record pivot methodology (SSH tunnel / Chisel / MSF route)
[ ] Note what credentials were used and where they came from
[ ] Record all internal subnets discovered at each stage

REPORT WRITING
[ ] Executive Summary: business language only, no commands or IPs
[ ] Findings ordered: Critical → High → Medium → Low → Informational
[ ] Each finding has: title, severity, CVSS, description, impact,
    evidence (with screenshot refs), reproduction steps, remediation
[ ] CVSS score calculated for every finding
[ ] Remediation is specific and actionable (not just "patch it")
[ ] Attack narrative explains the full kill chain as a story
[ ] Appendices include raw scan output and tool versions used
[ ] Proofread for spelling, consistency, professional tone
[ ] Export as PDF before submission

SEVERITY SELF-CHECK
[ ] RCE without auth → Critical (9.0+)
[ ] RCE with auth / LPE from SYSTEM → High (7.0–8.9)
[ ] Info disclosure / auth bypass → Medium (4.0–6.9)
[ ] Weak config / hardening gap → Low (0.1–3.9)
[ ] Best practice / defence-in-depth → Informational
```

---

## Remediation Timeframe Guidelines

```
Severity    Recommended Remediation Window
────────────────────────────────────────────────
Critical    Immediate / within 24–72 hours
High        Within 2 weeks
Medium      Within 30 days
Low         Within 90 days
Info        Next scheduled review cycle
```

---

## Common Vulnerability Classes Reference

| Vulnerability Class | OWASP / CWE | Typical Severity |
|---------------------|-------------|-----------------|
| Remote Code Execution | CWE-78 | Critical |
| SQL Injection | OWASP A03, CWE-89 | High–Critical |
| Broken Authentication | OWASP A07, CWE-287 | High |
| Sensitive Data Exposure | OWASP A02, CWE-200 | Medium–High |
| Local File Inclusion | CWE-98 | High |
| Cross-Site Scripting | OWASP A03, CWE-79 | Medium |
| IDOR | OWASP A01, CWE-639 | Medium–High |
| SSRF | OWASP A10, CWE-918 | High |
| XXE | OWASP A05, CWE-611 | High |
| Privilege Escalation | CWE-269 | High–Critical |
| Default Credentials | CWE-1392 | Medium–High |
| Unpatched Software | CWE-1104 | Varies by CVE |
| Weak Encryption | CWE-326 | Medium |
| Missing Auth on Function | OWASP A01, CWE-306 | High |

---

*eCPPTv3 Cheatsheet — 08_reporting_methodology.md*
*Cheatsheet series complete: 01–08*
