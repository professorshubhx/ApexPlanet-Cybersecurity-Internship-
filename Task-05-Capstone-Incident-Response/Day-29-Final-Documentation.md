# Day 29 — Final Documentation & Capstone Report
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## What We're Completing Today

Today is the final documentation day. Everything built across Days 24–28 gets compiled into a professional Capstone Project Report. This is the primary deliverable of Task 5 and the entire internship.

```
Day 24  Project Planning       → Executive Summary, Scope, Architecture
Day 25  ELK Stack Setup        → Implementation details, config
Day 26  Log Ingestion          → Data sources, pipeline config, indices
Day 27  Kibana Dashboards      → Visualization details, screenshots
Day 28  Incident Response      → IR simulation, timeline, findings
Day 29  Final Report           → Compile everything into one document
```

---

## Final Documentation Checklist

Before writing the report, confirm everything is in place:

```bash
# All services running
for svc in elasticsearch logstash kibana filebeat; do
  echo -n "$svc: "
  sudo systemctl is-active $svc
done

# Data in Elasticsearch
curl -s "http://localhost:9200/_cat/indices?v" | grep logs-

# Kibana accessible
curl -s http://localhost:5601 | head -3

# Screenshots taken
ls ~/screenshots/
```

---

## Screenshots to Take Before Writing Report

Take these screenshots in Kibana and terminal before compiling the report:

```
1.  ELK Stack all services running     (systemctl status output)
2.  Elasticsearch curl response        (cluster name: siem-lab, node: kali-node-1)
3.  Kibana home page                   (http://localhost:5601)
4.  Elasticsearch indices list         (curl _cat/indices showing logs-auth, logs-apache)
5.  Kibana Discover - auth logs        (showing SSH failed login events)
6.  Kibana Discover - apache logs      (showing attack events with attack_type field)
7.  Security Overview dashboard        (full dashboard view)
8.  Authentication Monitor dashboard   (showing brute force data)
9.  Web Security dashboard             (showing attack distribution)
10. Incident Response - blocked IP     (iptables -L showing block rule)
11. Post-eradication Nmap scan         (fewer open ports)
12. auth.log entries                   (grep "Failed password" output)
```

---

## Capstone Project Report Structure

### Section 1 — Executive Summary

```
Project:     Mini SIEM Implementation with ELK Stack
Author:      Professorshubhx
Academy:     Defronix Cybersecurity Academy
Mentor:      Nitesh Singh Sir
Internship:  ApexPlanet Software Pvt. Ltd.
Timeline:    Days 49–60 (12 days)
Date:        June 2026

Summary:
This capstone project implements a functional Security Information and
Event Management (SIEM) system using the open-source ELK Stack
(Elasticsearch 8.x, Logstash, Kibana, Filebeat) on Kali Linux.

The system collects and analyzes security logs from an isolated lab
environment, automatically detects attack patterns including SSH brute
force, SQL injection, path traversal, and web scanner activity, and
presents findings through real-time Kibana dashboards.

An incident response simulation was conducted to validate the SIEM's
detection capabilities, resulting in successful identification of
multiple attack vectors, containment of the threat, and complete
documentation of the incident lifecycle.

Key Achievements:
- ELK Stack fully operational on Kali Linux
- 3 Logstash pipelines processing auth, Apache, and Nmap logs
- 4 Kibana dashboards with 15+ visualizations
- Automatic detection of SSH brute force, SQLi, LFI, scanners
- Complete incident response simulation documented
- System hardened post-incident
```

---

### Section 2 — Project Scope and Objectives

```
Objectives:
1. Deploy a functional SIEM using open-source tools
2. Ingest and parse multiple log sources
3. Create dashboards for real-time security monitoring
4. Detect known attack patterns automatically
5. Simulate and document a complete incident response

In Scope:
  ELK Stack installation and configuration
  Log collection: auth.log, Apache access logs, Nmap XML
  Dashboard creation: 4 dashboards, 15+ visualizations
  Incident Response simulation against Metasploitable2
  System hardening post-incident

Out of Scope:
  Production deployment
  Windows event log collection
  Cloud SIEM integration
  Commercial Elastic features

Lab Environment:
  Attacker/SIEM: Kali Linux — 192.168.56.101
  Target:        Metasploitable2 — 192.168.56.104
  Network:       Host-Only 192.168.56.0/24 (isolated)
```

---

### Section 3 — Architecture and Design

```
NETWORK DIAGRAM:
┌─────────────────────────────────────────────┐
│         Host-Only: 192.168.56.0/24          │
│                                             │
│  ┌──────────────┐      ┌──────────────┐    │
│  │  Kali Linux  │      │Metasploitable│    │
│  │192.168.56.101│◄────►│192.168.56.104│    │
│  │              │      │              │    │
│  │ Elasticsearch│      │ FTP (21)     │    │
│  │ Logstash     │      │ SSH (22)     │    │
│  │ Kibana       │      │ HTTP (80)    │    │
│  │ Filebeat     │      │ MySQL (3306) │    │
│  └──────────────┘      └──────────────┘    │
└─────────────────────────────────────────────┘

ELK DATA FLOW:
Log Files → Filebeat → Logstash (parse) → Elasticsearch → Kibana

LOGSTASH PIPELINES:
auth-log.conf    /var/log/auth.log    → logs-auth-*
apache-log.conf  /var/log/apache2/    → logs-apache-*
nmap-log.conf    /root/nmap-*.xml     → logs-nmap-*
```

---

### Section 4 — Implementation

#### 4.1 ELK Stack Installation

```
Elasticsearch 8.x
  - Installed via Elastic APT repository
  - Configured: cluster.name=siem-lab, node.name=kali-node-1
  - Security disabled for lab environment
  - Running on: http://localhost:9200

Logstash 8.x
  - 3 pipeline configurations created
  - Processes auth, Apache, and Nmap logs
  - Running on: localhost:5044 (Beats input)

Kibana 8.x
  - Configured: server.host=0.0.0.0, port=5601
  - Connected to Elasticsearch
  - Running on: http://localhost:5601

Filebeat 8.x
  - Configured to ship auth.log and Apache logs
  - Output: Logstash on localhost:5044
```

#### 4.2 Log Parsing — Grok Patterns

```
Auth Log Parse Pattern:
%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:hostname}
%{WORD:program}(?:\[%{POSINT:pid}\])?:
%{GREEDYDATA:log_message}

Fields extracted:
  timestamp     Event time
  hostname      Machine name
  program       sshd, su, sudo
  event_type    ssh_failed_login / ssh_successful_login
  src_ip        Attacker IP address
  failed_user   Username attempted
  severity      high / medium / info

Apache Log Parse Pattern:
%{COMBINEDAPACHELOG}

Additional fields added:
  attack_type   possible_sqli_xss / path_traversal / scanner_detected
  severity      high / medium
```

#### 4.3 Attack Detection Rules

```
Rule 1: SQL/XSS Detection
  Pattern: UNION|SELECT|INSERT|DROP|script|alert in request URL
  Field:   attack_type = possible_sqli_xss
  Severity: high

Rule 2: Path Traversal Detection
  Pattern: ../ in request URL
  Field:   attack_type = path_traversal
  Severity: high

Rule 3: Scanner Detection
  Pattern: nikto|sqlmap|nmap|masscan in User-Agent
  Field:   attack_type = scanner_detected
  Severity: medium

Rule 4: SSH Failed Login
  Pattern: "Failed password" in log message
  Field:   event_type = ssh_failed_login
  Severity: medium

Rule 5: SSH Successful Login
  Pattern: "Accepted password" in log message
  Field:   event_type = ssh_successful_login
  Severity: info
```

---

### Section 5 — Kibana Dashboards

```
Dashboard 1: Security Overview
  Panels: Total Events (Metric), Events by Severity (Pie),
          Top Attacking IPs (Bar), Events Over Time (Line),
          Event Types Summary (Table)

Dashboard 2: Authentication Monitor
  Panels: SSH Login Attempts Over Time (Bar),
          Top Brute Force IPs (Table),
          Failed Login Count (Metric),
          Targeted Usernames (Pie)

Dashboard 3: Web Application Security
  Panels: Web Attack Distribution (Donut),
          HTTP Response Codes (Bar),
          Top Attacked URLs (Table),
          Web Traffic Over Time (Area),
          Scanner Detections (Metric)

Dashboard 4: Network Activity
  Panels: Open Ports Discovered (Bar),
          Services Tag Cloud
```

---

### Section 6 — Incident Response Simulation

#### Incident Summary

```
Incident ID:    IR-2026-001
Date:           June 2026
Severity:       Critical
Type:           Active Intrusion / Unauthorized Root Access
Attacker IP:    192.168.56.101
Target IP:      192.168.56.104
Duration:       Simulated — approximately 30 minutes
```

#### Attack Timeline

```
T+00:00  Nmap reconnaissance scan initiated
         192.168.56.101 → 192.168.56.104
         26 open ports discovered

T+02:00  SSH brute force started (Hydra)
         14,344,402 password attempts against msfadmin
         No account lockout — attack proceeds unhindered

T+10:00  SIEM Alert triggered
         SSH brute force threshold exceeded
         >10 failed logins per minute from 192.168.56.101

T+12:00  vsftpd 2.3.4 exploit launched (Metasploit)
         CVE-2011-2523 backdoor triggered
         Root shell obtained: uid=0(root)

T+14:00  Samba exploit launched
         CVE-2007-2447 usermap_script
         Second root shell + Meterpreter session

T+16:00  Post-exploitation
         sysinfo, getuid, system enumeration
         /etc/passwd read

T+20:00  SOC Analyst detects incident in Kibana
         Multiple dashboards showing attack activity
         Incident declared — IR process initiated

T+22:00  Containment
         Attacker IP blocked: iptables -I INPUT 1 -s 192.168.56.101 -j DROP
         Active sessions killed
         Evidence preserved

T+25:00  Eradication
         vsftpd removed: apt remove vsftpd
         Telnet disabled
         Port 1524 closed
         Credentials reset

T+28:00  Recovery
         Firewall hardened — default deny policy
         System updates applied
         Post-eradication Nmap scan — fewer open ports

T+30:00  Incident closed
         Post-Incident Report drafted
```

#### Detection Evidence from Logs

```bash
# Failed login count from attacker IP
grep "Failed password" /var/log/auth.log | \
  grep "192.168.56.101" | wc -l
# Result: 1,247 failed attempts

# Attack types detected in Apache
curl -s "http://localhost:9200/logs-apache-*/_search" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"exists":{"field":"attack_type"}},"size":0,"aggs":{"types":{"terms":{"field":"attack_type.keyword"}}}}'
# Result: possible_sqli_xss: 24, path_traversal: 8, scanner_detected: 6
```

---

### Section 7 — Findings and Vulnerabilities

| # | Finding | Severity | CVE |
|---|---------|----------|-----|
| 1 | vsftpd 2.3.4 backdoor | Critical | CVE-2011-2523 |
| 2 | Samba RCE | Critical | CVE-2007-2447 |
| 3 | No SSH rate limiting or lockout | High | — |
| 4 | 26 unnecessary open ports | High | — |
| 5 | Telnet running (plaintext) | High | — |
| 6 | Root bindshell on port 1524 | Critical | — |
| 7 | No firewall rules | Critical | — |
| 8 | MySQL accessible from network | High | — |

---

### Section 8 — Mitigations Applied

| Finding | Mitigation | Status |
|---------|-----------|--------|
| vsftpd backdoor | Removed vsftpd package | Done |
| No SSH lockout | Firewall + rate limiting | Done |
| Excess open ports | iptables default-deny | Done |
| Telnet running | Disabled service | Done |
| No firewall | iptables ruleset applied | Done |
| SSH root login | PermitRootLogin no | Done |

---

### Section 9 — SIEM Effectiveness Assessment

```
What the SIEM successfully detected:
  SSH brute force attack       (auth.log → Logstash → Kibana alert)
  SQL injection attempts       (Apache log pattern matching)
  Path traversal attempts      (LFI detection rule triggered)
  Web scanner activity         (User-Agent detection)
  Post-exploitation activity   (unusual process and connection)

What could be improved:
  Lower alert threshold for faster detection
  Add Metasploit activity detection rules
  Integrate firewall logs (iptables LOG rules)
  Add network flow analysis
  Implement email/Slack alerting
  Add Windows event log support
```

---

### Section 10 — Recommendations

```
Priority 1 — Immediate:
  Patch all systems — vsftpd, Samba, Apache, MySQL
  Implement fail2ban for SSH
  Apply default-deny firewall policy
  Disable unused services

Priority 2 — Short Term:
  Implement centralized logging for all systems
  Configure SIEM alerting (email/Slack notifications)
  Add Windows event log collection
  Regular vulnerability scanning schedule

Priority 3 — Long Term:
  Migrate to cloud SIEM (Elastic Cloud, Splunk Cloud)
  Implement SOAR for automated response
  Regular red team exercises
  SOC analyst training program
```

---

### Section 11 — Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| Elasticsearch | 8.19.16 | Data storage and search |
| Logstash | 8.x | Log parsing and ingestion |
| Kibana | 8.x | Visualization and dashboards |
| Filebeat | 8.x | Log shipping |
| Nmap | 7.99 | Network scanning |
| Metasploit | 6.x | Exploitation framework |
| Hydra | 9.5 | Password brute forcing |
| Wireshark | 4.x | Packet analysis |
| iptables | — | Firewall management |
| Kali Linux | Rolling | Attacker/SIEM platform |

---

### Section 12 — Lessons Learned

```
Technical:
  ELK Stack is powerful but resource-intensive — needs 4GB+ RAM
  Grok pattern debugging requires patience — use grokdebug.herokuapp.com
  Log standardization is critical — different formats need different parsers
  Index patterns must be created before Kibana can visualize data

Security:
  Unpatched systems are trivially exploited — time from scan to root was 12 minutes
  SIEM is only as good as its log sources — garbage in, garbage out
  Detection rules need tuning — too sensitive = alert fatigue
  Incident response speed matters — every minute = more damage

Process:
  Documentation during incident is critical — things move fast
  Evidence preservation must happen before containment
  Post-incident report should be written within 24 hours while memory is fresh
```

---

### Section 13 — Appendices

```
Appendix A: ELK Stack Configuration Files
  - elasticsearch.yml
  - kibana.yml
  - filebeat.yml
  - auth-log.conf (Logstash)
  - apache-log.conf (Logstash)

Appendix B: Screenshots
  (see screenshots/ folder)

Appendix C: Raw Log Excerpts
  - auth.log brute force evidence
  - Apache attack log entries
  - Nmap scan output

Appendix D: Evidence Hashes
  - SHA256 of preserved log files

Appendix E: Nmap Scan Reports
  - Pre-hardening scan (26 ports)
  - Post-hardening scan (reduced ports)
```

---

## Post-Incident Report (Separate Document)

```
POST-INCIDENT REPORT
Incident ID: IR-2026-001
Date: June 2026
Analyst: Professorshubhx

EXECUTIVE SUMMARY:
On June 2, 2026, the SIEM detected an active intrusion against
Metasploitable2 (192.168.56.104). The attacker at 192.168.56.101
performed reconnaissance, SSH brute force, and exploited two
critical vulnerabilities (vsftpd CVE-2011-2523 and Samba
CVE-2007-2447) to gain root access. The incident was contained
within 8 minutes of detection. Affected services were hardened
and all vulnerabilities addressed.

TIMELINE: [See Section 6 above]

ROOT CAUSE:
Unpatched vsftpd 2.3.4 (2011 vulnerability) and Samba 3.0.20
(2007 vulnerability) were running without firewall protection.
No account lockout policy allowed unrestricted brute force.

IMPACT:
Root-level access obtained on target system.
/etc/passwd read. System information enumerated.
No production data affected (isolated lab environment).

CORRECTIVE ACTIONS:
1. Removed vsftpd — replaced with SFTP
2. Upgraded Samba
3. Implemented iptables default-deny firewall
4. Disabled Telnet, r-services, unnecessary ports
5. Applied PermitRootLogin no in sshd_config
6. Installed fail2ban for SSH protection

PREVENTION:
Regular vulnerability scanning (monthly minimum)
Patch management process — critical patches within 48 hours
SIEM monitoring with automated alerting
Quarterly penetration testing
```

---

## GitHub Repository Final Structure

```
ApexPlanet-Cybersecurity-Internship/
├── README.md
├── Task-01-Foundation-and-Environment-Setup/
├── Task-02-Network-Security-and-Scanning/
├── Task-03-Web-Application-Security/
├── Task-04-Exploitation-and-System-Security/
└── Task-05-Capstone-Incident-Response/
    ├── README.md
    ├── Day-24-Project-Planning.md
    ├── Day-25-ELK-Stack-Setup.md
    ├── Day-26-Log-Ingestion.md
    ├── Day-27-Kibana-Dashboards.md
    ├── Day-28-Incident-Response.md
    ├── Day-29-Final-Documentation.md      ← This file
    ├── Capstone-Project-Report.md
    ├── Post-Incident-Report.md
    └── screenshots/
        ├── elk-services-running.png
        ├── elasticsearch-response.png
        ├── kibana-home.png
        ├── elasticsearch-indices.png
        ├── kibana-auth-logs.png
        ├── kibana-apache-logs.png
        ├── dashboard-security-overview.png
        ├── dashboard-auth-monitor.png
        ├── dashboard-web-security.png
        ├── iptables-block.png
        ├── nmap-post-hardening.png
        └── auth-log-brute-force.png
```

---

## Practice Tasks

1. Take all 12 screenshots listed at the top of this file
2. Write your Executive Summary in your own words — 3-4 sentences
3. Complete Section 6 (Incident Timeline) with your actual timestamps
4. Fill in the actual document counts from your Elasticsearch indices
5. Export Kibana dashboard as PDF: Dashboard → Share → PDF Reports
6. Save Logstash config files in the GitHub repo under `configs/`
7. Write the Post-Incident Report using the template above
8. Update the main repo README.md — mark Task 5 as Complete
9. Record the 12-minute final video using the script in the next section
10. Final submission: GitHub repo link + video link

---

## 12-Minute Final Video Script Outline

```
00:00 – 00:45  Introduction
               "Hi, I'm Professorshubhx. This is my Capstone Project
               for the ApexPlanet Cybersecurity Internship —
               a Mini SIEM built with the ELK Stack."

00:45 – 02:00  Project Overview
               Show the network diagram
               Explain what SIEM is and why it matters for SOC

02:00 – 04:00  ELK Stack Demo
               Show Elasticsearch running (curl localhost:9200)
               Show Kibana home page
               Show indices with log data

04:00 – 06:30  Kibana Dashboards
               Walk through Security Overview dashboard
               Show Authentication Monitor — brute force data
               Show Web Security — attack types

06:30 – 08:30  Live Detection Demo
               Run a quick Hydra brute force
               Watch it appear in Kibana in real time
               Show the auth log entries

08:30 – 10:00  Incident Response
               Explain IR phases
               Show containment — iptables block
               Show eradication — service removal

10:00 – 11:00  Findings Summary
               Show vulnerability table
               Show mitigations applied

11:00 – 12:00  Conclusion
               "This project covers the full SOC analyst workflow —
               from attack detection through incident response.
               All notes and reports on GitHub. Link in description.
               Thanks to Nitesh Singh Sir at Defronix Academy
               and ApexPlanet for this internship."
```

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 29 Complete | Task 5 Complete | ApexPlanet Internship Complete*
*Total: 5 Tasks | 29 Days of Notes | Full Penetration Testing + SIEM + Incident Response*
