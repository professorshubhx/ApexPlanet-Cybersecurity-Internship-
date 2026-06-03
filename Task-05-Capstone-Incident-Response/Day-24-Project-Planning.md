# Day 24 — Capstone Project Planning
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## Project Overview

**Project Title:** Mini SIEM Implementation with ELK Stack
**Objective:** Build a functional Security Information and Event Management system using the ELK Stack (Elasticsearch, Logstash, Kibana) to collect, parse, correlate, and visualize security logs from our lab environment.

This project brings together everything from Tasks 1–4 — the lab setup, network scanning, exploitation, and incident response — into a single unified security monitoring platform. Instead of just attacking, we're now building the defense.

---

## Why SIEM for a SOC L1 Role

A SOC L1 analyst spends their entire day inside a SIEM. Every alert, every investigation, every escalation starts in the SIEM dashboard. Building one from scratch demonstrates:

```
Technical depth      You understand how logs flow, get parsed, and get visualized
Defensive mindset    You can think like a defender, not just an attacker
Tool familiarity     ELK Stack is used in real SOC environments globally
Integration skills   Connecting multiple data sources into one platform
```

---

## Project Scope

```
In Scope:
  ELK Stack installation on Kali Linux
  Log collection from:
    - /var/log/auth.log       (SSH brute force detection)
    - /var/log/apache2/       (Web attack detection — DVWA traffic)
    - Nmap XML output         (Scan activity)
    - Metasploit logs         (Exploitation attempts)
  Kibana dashboard creation
  Alert configuration for suspicious activity
  Incident Response simulation using collected logs

Out of Scope:
  Production deployment
  Windows log collection (EVTX)
  Cloud integration
  Paid Elastic features
```

---

## Tools and Technologies

| Tool | Version | Purpose |
|------|---------|---------|
| Elasticsearch | 8.x | Data storage and search engine |
| Logstash | 8.x | Log collection, parsing, forwarding |
| Kibana | 8.x | Visualization and dashboard UI |
| Filebeat | 8.x | Lightweight log shipper |
| Kali Linux | Rolling | Host OS for ELK Stack |
| Metasploitable2 | — | Attack target, log source |
| Nmap | 7.99 | Network scanning, XML output |

---

## Timeline

| Day | Activity |
|-----|---------|
| Day 24 | Project planning, network diagram, scope definition |
| Day 25 | ELK Stack installation and configuration |
| Day 26 | Log ingestion — auth.log, Apache, Nmap XML |
| Day 27 | Kibana dashboards — SSH attacks, web traffic, scan activity |
| Day 28 | Incident Response simulation — detect, contain, eradicate |
| Day 29 | Final documentation, Capstone Report, Post-Incident Report |
| Day 30 | Review, cleanup, GitHub organization, video prep |

---

## Network Diagram

```
+--------------------------------------------------+
|           Host-Only Network: 192.168.56.0/24     |
|                                                  |
|  +-------------------+    +-------------------+  |
|  |   Kali Linux      |    |  Metasploitable2  |  |
|  |  192.168.56.101   |    |  192.168.56.104   |  |
|  |                   |    |                   |  |
|  |  [ELK Stack]      |    |  [Target]         |  |
|  |  Elasticsearch    |◄───|  FTP (21)         |  |
|  |  Logstash         |    |  SSH (22)         |  |
|  |  Kibana           |    |  HTTP (80)        |  |
|  |  Filebeat         |    |  MySQL (3306)     |  |
|  |                   |    |  Samba (445)      |  |
|  |  [Attack Tools]   |───►|                   |  |
|  |  Nmap             |    |  [Log Sources]    |  |
|  |  Metasploit       |    |  auth.log         |  |
|  |  Hydra            |    |  apache2/         |  |
|  |  Burp Suite       |    |  syslog           |  |
|  +-------------------+    +-------------------+  |
|                                                  |
+--------------------------------------------------+

Log Flow:
Metasploitable2 logs ──► Filebeat ──► Logstash ──► Elasticsearch ──► Kibana
Kali attack logs    ──► Filebeat ──► Logstash ──► Elasticsearch ──► Kibana
Nmap XML output     ──► Logstash ──► Elasticsearch ──► Kibana
```

---

## Data Flow Architecture

```
DATA SOURCES          COLLECTION      PROCESSING      STORAGE         VISUALIZATION
─────────────         ──────────      ──────────      ───────         ─────────────
/var/log/auth.log ──► Filebeat    ──► Logstash    ──► Elasticsearch ──► Kibana
/var/log/apache2/ ──► Filebeat    ──►   │          ►  Index: logs-*      Dashboard
Nmap XML output   ──► Logstash   ─┘   Filter      ►  Index: nmap-*      Alerts
Metasploit logs   ──► Filebeat       Enrich                              Discover
                                     Parse
                                     Timestamp
```

---

## Log Sources and What They Tell Us

### /var/log/auth.log — Authentication Events

```
What it logs:
  SSH login attempts (success and failure)
  sudo usage
  User account changes
  PAM authentication events

What we can detect:
  Brute force attacks (many failed logins from same IP)
  Successful logins after failures (credential stuffing)
  Root login attempts
  Lateral movement via SSH

Sample log line:
  May 31 23:44:33 kali sshd[1234]: Failed password for msfadmin
  from 192.168.56.101 port 45678 ssh2
```

### /var/log/apache2/access.log — Web Traffic

```
What it logs:
  Every HTTP request to the web server
  Source IP, URL, response code, user agent, bytes

What we can detect:
  SQL injection attempts (UNION, SELECT in URL)
  XSS attempts (<script> in parameters)
  Directory traversal (../../etc/passwd)
  Brute force on login pages (many POST to login.php)
  Scanner activity (Nikto, Dirb signatures in user-agent)

Sample log line:
  192.168.56.101 - - [01/Jun/2026:11:30:45] "GET
  /dvwa/vulnerabilities/sqli/?id=1'+UNION+SELECT+user(),database()--+-
  HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
```

### Nmap XML Output — Network Reconnaissance

```
What it logs:
  All scanned ports and their states
  Service versions detected
  OS fingerprint
  Script scan results

What we can detect:
  Port scan activity
  Vulnerability discovery attempts
  Service enumeration
```

---

## SIEM Use Cases — What We Will Detect

These are the detection rules we will build in Kibana:

```
Use Case 1: SSH Brute Force Detection
  Rule:     More than 5 failed SSH logins from same IP in 60 seconds
  Source:   auth.log
  Severity: High
  Response: Alert + log attacker IP

Use Case 2: Successful Login After Brute Force
  Rule:     Successful SSH login from IP that had previous failures
  Source:   auth.log
  Severity: Critical
  Response: Immediate alert — likely compromised account

Use Case 3: SQL Injection Attempt
  Rule:     HTTP request containing UNION, SELECT, OR 1=1, --, DROP
  Source:   apache2/access.log
  Severity: High
  Response: Alert + block IP at firewall

Use Case 4: Web Scanner Detection
  Rule:     User-Agent containing nikto, sqlmap, masscan, nmap
  Source:   apache2/access.log
  Severity: Medium
  Response: Log and alert

Use Case 5: Port Scan Detection
  Rule:     Single IP connecting to more than 10 ports in 30 seconds
  Source:   Firewall logs / iptables LOG
  Severity: Medium
  Response: Alert + investigate source IP

Use Case 6: Root Login Attempt
  Rule:     Any authentication attempt with username "root" via SSH
  Source:   auth.log
  Severity: High
  Response: Immediate alert
```

---

## Elasticsearch Index Structure

We will create separate indices for different log types:

```
logs-auth-*         Authentication logs from auth.log
logs-apache-*       Web server access logs
logs-nmap-*         Nmap scan results
logs-system-*       General syslog
logs-firewall-*     iptables firewall logs
```

---

## Kibana Dashboards Planned

```
Dashboard 1: Security Overview
  - Total events in last 24h (metric)
  - Events by severity (pie chart)
  - Top 10 source IPs (bar chart)
  - Events over time (line chart)

Dashboard 2: Authentication Monitoring
  - Failed vs successful logins (bar chart)
  - Top attacking IPs (table)
  - Login attempts over time (area chart)
  - Geographic source map (if public IPs)

Dashboard 3: Web Application Security
  - HTTP response codes (pie chart)
  - Top requested URLs (table)
  - Suspicious patterns detected (table)
  - Traffic over time (line chart)

Dashboard 4: Network Activity
  - Nmap scan findings (table)
  - Open ports discovered (bar chart)
  - Vulnerability count by severity (bar chart)
```

---

## Incident Response Plan

For the simulation, we will:

```
Phase 1: Detection
  Review Kibana dashboard
  Identify suspicious activity in logs
  Correlate events across multiple sources

Phase 2: Analysis
  Determine attack type and scope
  Identify attacker IP and techniques
  Timeline of events

Phase 3: Containment
  Block attacker IP via iptables
  Disable compromised account

Phase 4: Eradication
  Remove any persistence mechanisms
  Patch exploited vulnerability

Phase 5: Recovery
  Restore services
  Verify clean state

Phase 6: Post-Incident Report
  Document the entire incident
  Lessons learned
  Improvements to detection
```

---

## Practice Tasks

1. Confirm Kali has at least 4GB RAM assigned in VirtualBox settings
2. Check available disk space: `df -h` — need at least 10GB free
3. Check Java is installed: `java -version` — ELK needs Java 11+
4. Draw the network diagram on paper or in draw.io — understand all connections
5. Read the 6 use cases above — understand what each one detects and why
6. Run a Hydra brute force against Metasploitable2 SSH to generate some auth.log entries: `hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.104 -t 4`
7. Browse DVWA and try some SQL injection — generate Apache access log entries
8. Check if auth.log has entries: `sudo tail -50 /var/log/auth.log`
9. Check Apache logs: `sudo tail -50 /var/log/apache2/access.log`
10. Create the GitHub folder: `Task-05-Capstone-Incident-Response/` with a placeholder README

---

## Quick Reference

```
ELK COMPONENTS                  LOG LOCATIONS
Elasticsearch  Port 9200        /var/log/auth.log
Logstash       Port 5044        /var/log/apache2/access.log
Kibana         Port 5601        /var/log/apache2/error.log
Filebeat       Agent            /var/log/syslog

DETECTION USE CASES             INDEX NAMING
SSH Brute Force                 logs-auth-YYYY.MM.DD
SQLi Detection                  logs-apache-YYYY.MM.DD
Web Scanner                     logs-nmap-YYYY.MM.DD
Port Scan                       logs-firewall-YYYY.MM.DD
Root Login Attempt
Successful post-brute login
```

---

## Resources

- Elastic Documentation: https://www.elastic.co/guide/
- ELK Stack Tutorial: https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-22-04
- Kibana Query Language: https://www.elastic.co/guide/en/kibana/current/kuery-query.html
- SIEM Use Cases: https://www.elastic.co/guide/en/security/current/detection-engine-overview.html
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 24 Complete | Next: Day 25 — ELK Stack Installation and Configuration*
