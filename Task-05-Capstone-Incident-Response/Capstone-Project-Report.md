# Mini SIEM Implementation and Incident Response Report
### ApexPlanet Internship – Task 05 Capstone Project

---

## Project Information

| Field | Details |
|---------|----------|
| Project Title | Mini SIEM Implementation and Incident Response |
| Internship | ApexPlanet Cybersecurity Internship |
| Student | Shubham Chaurasiya |
| Technologies Used | Elasticsearch, Logstash, Kibana, Filebeat (ELK Stack) |
| Environment | Kali Linux Virtual Machine |
| Project Duration | Day 24 – Day 29 |
| Submission | Task 05 – Capstone Project |

---

# Executive Summary

This capstone project focused on designing and implementing a lightweight Security Information and Event Management (SIEM) solution using the ELK Stack. The objective was to simulate the responsibilities of a Security Operations Center (SOC) analyst by collecting logs from multiple sources, identifying suspicious activities, visualizing security events through dashboards, and documenting the incident response process.

The implemented solution successfully ingested authentication logs, web application events, and Nmap reconnaissance data using Filebeat and Logstash. The collected data was indexed into Elasticsearch and visualized through Kibana dashboards designed for monitoring security activities.

The project demonstrates practical understanding of log analysis, detection engineering, dashboard creation, and incident response documentation within a controlled lab environment.

---

# Project Objectives

The primary objectives of this project were:

- Deploy and configure the ELK Stack.
- Collect logs from multiple security-relevant sources.
- Build log ingestion pipelines using Filebeat and Logstash.
- Create SOC-style dashboards using Kibana.
- Detect suspicious activities and reconnaissance attempts.
- Simulate incident investigation workflows.
- Document findings and response procedures.

---

# Lab Environment

## Operating Environment

| Component | Details |
|-----------|-----------|
| Host Operating System | Kali Linux |
| Virtualization Platform | VirtualBox |
| Elasticsearch Version | 8.19.16 |
| Logstash Version | 8.19.16 |
| Kibana Version | 8.19.16 |
| Filebeat Version | 8.19.16 |

---

## Log Sources

The following log sources were integrated into the SIEM solution:

### Authentication Logs

Collected from:

- `/var/log/auth.log`

Used for:

- Failed login monitoring
- User authentication analysis
- Brute-force detection

---

### Nmap Reconnaissance Logs

Collected from:

- XML output generated during Nmap scans.

Used for:

- Open port analysis
- Service discovery
- Reconnaissance activity monitoring

---

### Web Application Logs

Collected from:

- Apache Access Logs

Used for:

- Scanner detection
- HTTP status monitoring
- Suspicious URL identification

---

# ELK Stack Architecture

The implemented architecture followed the pipeline shown below:

```
Log Sources
     │
     ▼
 Filebeat
     │
     ▼
 Logstash
(Filtering & Parsing)
     │
     ▼
 Elasticsearch
(Indexing & Storage)
     │
     ▼
 Kibana
(Visualization & Investigation)
```

Detailed architecture diagram is available in:

```
network-diagram.md
```

---

# Log Ingestion Pipeline

## Filebeat

Filebeat was configured to monitor and forward logs from multiple sources.

Key responsibilities included:

- Monitoring designated log paths.
- Tagging incoming events.
- Forwarding events to Logstash.

Examples of monitored data included:

- Authentication events
- Apache access logs
- Nmap XML scan outputs

---

## Logstash

Logstash served as the parsing and enrichment layer.

Implemented functions included:

- Event classification.
- Field extraction.
- Tag assignment.
- Detection rule application.
- Event enrichment.

Examples of generated fields:

- Severity
- Event Type
- Service Name
- Port Information

---

## Elasticsearch

Elasticsearch stored processed events within dedicated indices.

Examples:

```
real-logs-*
security-logs-*
web-logs-*
```

This enabled efficient searching and visualization through Kibana.

---

# Kibana Dashboards

Four dashboards were developed to simulate SOC analyst workflows.

---

## 1. Security Logs Dashboard

Purpose:

Monitor overall security events and identify abnormal activities.

Features:

- Total security event count.
- Failed login monitoring.
- Scanner detection.
- Severity distribution.

### Dashboard Screenshot

[![Security Logs Dashboard](screenshots/security-dashboard.png)](screenshots/security-dashboard.png)

---

## 2. Authentication Monitor Dashboard

Purpose:

Investigate authentication-related incidents.

Features:

- Failed SSH login attempts.
- Targeted usernames.
- Source IP analysis.
- Login attempt timeline.

### Dashboard Screenshot

[![Authentication Monitor](screenshots/auth-dashboard.png)](screenshots/auth-dashboard.png)

---

## 3. Web Application Security Dashboard

Purpose:

Monitor web application activity and identify suspicious requests.

Features:

- Attack distribution.
- HTTP response analysis.
- Frequently targeted URLs.
- Scanner detection.

### Dashboard Screenshot

[![Web Application Security](screenshots/web-dashboard.png)](screenshots/web-dashboard.png)

---

## 4. Nmap Recon Dashboard

Purpose:

Analyze reconnaissance activities against monitored systems.

Features:

- Open port identification.
- Service discovery.
- Reconnaissance timeline.
- Critical service detection.

### Dashboard Screenshot

[![Nmap Recon Dashboard](screenshots/nmap-dashboard.png)](screenshots/nmap-dashboard.png)

---

# Findings and Analysis

Analysis of collected logs revealed several noteworthy observations.

## Authentication Findings

- Multiple failed authentication attempts were identified.
- Repeated targeting of specific usernames was observed.
- Source IP analysis indicated brute-force simulation activity.

---

## Web Security Findings

- Scanner-generated requests were detected.
- Enumeration attempts targeting common endpoints were identified.
- Multiple HTTP status codes highlighted both successful and failed requests.

---

## Reconnaissance Findings

Nmap scan analysis identified numerous exposed services including:

- FTP
- SSH
- HTTP
- NetBIOS
- MySQL
- Java RMI
- IRC

The reconnaissance dashboard provided visibility into the attack surface of the target environment.

---

# Incident Response Workflow

The following workflow was adopted during investigations:

1. Event Detection.
2. Initial Triage.
3. Log Correlation.
4. Indicator Identification.
5. Impact Assessment.
6. Containment Recommendations.
7. Documentation of Findings.

A detailed incident investigation is documented in:

```
Post-Incident-Report.md
```

---

# Challenges Encountered

During implementation, several technical challenges were encountered:

- Filebeat registry synchronization issues.
- Logstash parsing failures.
- Elasticsearch indexing delays.
- Resource constraints within the virtual environment.
- Dashboard visualization tuning.

These challenges were resolved through systematic troubleshooting and validation.

---

# Skills Demonstrated

This project helped strengthen practical skills in:

- Security Monitoring
- SIEM Operations
- ELK Stack Administration
- Log Analysis
- Detection Engineering
- Incident Investigation
- Dashboard Development
- Technical Documentation

---

# Recommendations

Future improvements may include:

- Sigma rule integration.
- Elastic Alerting.
- Email notifications.
- Threat intelligence enrichment.
- GeoIP analysis.
- Automated incident workflows.
- Integration with additional log sources.

---

# Conclusion

This capstone project successfully demonstrated the implementation of a functional SIEM solution using the ELK Stack within a laboratory environment. By integrating multiple log sources, developing dashboards, and simulating incident response activities, the project provided hands-on experience aligned with real-world SOC operations.

The knowledge gained through this implementation strengthened understanding of security monitoring, event correlation, investigation methodologies, and the importance of structured incident response processes.

The project represents a significant milestone in transitioning theoretical cybersecurity concepts into practical defensive capabilities.
