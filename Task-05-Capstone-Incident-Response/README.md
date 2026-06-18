# 🛡️ Task 05 – Capstone Incident Response Project

> Building a mini Security Operations Center (SOC) using the ELK Stack to collect, analyze, visualize, and investigate security events generated from a controlled cyber security lab environment.

---

## 📌 Project Overview

This capstone project demonstrates the complete Security Information and Event Management (SIEM) lifecycle using the **ELK Stack (Elasticsearch, Logstash, Kibana)**.

A vulnerable lab environment was created using **Kali Linux** and **Metasploitable 2**. Multiple attack simulations and reconnaissance activities were performed, logs were collected through **Filebeat**, processed by **Logstash**, indexed into **Elasticsearch**, and visualized using **Kibana dashboards**.

The project concludes with an incident investigation and professional documentation similar to real-world SOC operations.

---

## 🎯 Objectives

- Deploy and configure the ELK Stack.
- Collect logs from multiple security sources.
- Simulate reconnaissance and attack activities.
- Build actionable SIEM dashboards.
- Detect suspicious activities using Kibana.
- Perform incident analysis and reporting.
- Document the complete investigation process.

---

# 🏗️ Lab Architecture

```
Attacker Machine
(Kali Linux)
       │
       │ Reconnaissance / Attack Simulation
       ▼
Target Machine
(Metasploitable 2)
       │
       │ Log Collection
       ▼
Filebeat
       │
       ▼
Logstash
       │
       ▼
Elasticsearch
       │
       ▼
Kibana Dashboards
```

---

## 🖥️ Environment Details

| Component | Details |
|-----------|-----------|
| Operating System | Kali Linux |
| Target Machine | Metasploitable 2 |
| Elasticsearch | 8.19.16 |
| Logstash | 8.19.16 |
| Kibana | 8.19.16 |
| Filebeat | 8.19.16 |
| Virtualization | VirtualBox |
| SIEM Platform | ELK Stack |

---

# 📂 Repository Structure

```text
Task-05-Capstone-Incident-Response/
├── README.md
├── Day-24-Project-Planning.md
├── Day-25-ELK-Stack-Setup.md
├── Day-26-Log-Ingestion.md
├── Day-27-SIEM-Dashboards.md
├── Day-28-Incident-Response.md
├── Day-29-Final-Documentation.md
├── Capstone-Project-Report.md
├── Post-Incident-Report.md
├── network-diagram.md
└── screenshots/
```

---

# 📅 Project Timeline

## Day 24 – Project Planning

- Defined project objectives.
- Designed the SOC architecture.
- Selected tools and technologies.
- Planned attack simulations.

➡️ [View Documentation](./Day-24-Project-Planning.md)

---

## Day 25 – ELK Stack Setup

- Installed Elasticsearch.
- Installed Logstash.
- Installed Kibana.
- Verified service connectivity.
- Configured the SIEM environment.

➡️ [View Documentation](./Day-25-ELK-Stack-Setup.md)

---

## Day 26 – Log Ingestion

Implemented log collection using Filebeat.

### Log Sources

- Nmap XML Scan Logs
- Authentication Logs
- Security Event Logs
- Web Activity Logs

➡️ [View Documentation](./Day-26-Log-Ingestion.md)

---

## Day 27 – SIEM Dashboards

Created multiple Kibana dashboards for security monitoring.

### Dashboards Developed

- Security Logs Dashboard
- Authentication Monitor
- Web Application Security Dashboard
- Nmap Recon Dashboard

➡️ [View Documentation](./Day-27-SIEM-Dashboards.md)

---

## Day 28 – Incident Response

Conducted investigation using collected evidence.

Activities included:

- Event triage
- Timeline analysis
- Detection validation
- Root cause analysis
- Incident containment recommendations

➡️ [View Documentation](./Day-28-Incident-Response.md)

---

## Day 29 – Final Documentation

Compiled all findings into professional reports.

Generated:

- Capstone Project Report
- Post Incident Report
- Network Diagram

➡️ [View Documentation](./Day-29-Final-Documentation.md)

---

# 📊 Dashboard Highlights

## Security Logs Dashboard

Monitors:

- Security events
- Failed login attempts
- Severity levels
- Scanner detections

### Screenshot

[![Security Logs Dashboard](screenshots/security-logs-dashboard.png)](screenshots/security-logs-dashboard.png)

---

## Authentication Monitor

Provides visibility into:

- SSH login attempts
- Failed authentications
- Targeted usernames
- Brute-force source IPs

### Screenshot

[![Authentication Dashboard](screenshots/auth-dashboard.png)](screenshots/auth-dashboard.png)

---

## Web Application Security Dashboard

Tracks:

- HTTP status codes
- Scanner activity
- Suspicious URLs
- Web traffic trends

### Screenshot

[![Web Security Dashboard](screenshots/web-dashboard.png)](screenshots/web-dashboard.png)

---

## Nmap Recon Dashboard

Displays:

- Open ports discovered
- Services identified
- Reconnaissance trends
- Critical services detected

### Screenshot

[![Nmap Dashboard](screenshots/nmap-dashboard.png)](screenshots/nmap-dashboard.png)

---

# 🚨 Incident Summary

During the investigation, the SIEM detected several suspicious activities:

### Reconnaissance Activity

- Extensive Nmap scans were identified.
- Multiple services and ports were enumerated.

### Authentication Events

- Failed SSH login attempts observed.
- Repeated username targeting patterns detected.

### Web Enumeration

- Scanner requests identified through HTTP logs.
- Abnormal URL requests recorded.

---

# 📑 Reports

## Capstone Project Report

Comprehensive documentation covering:

- Architecture
- Methodology
- Findings
- Dashboards
- Lessons learned

➡️ [Open Report](./Capstone-Project-Report.md)

---

## Post Incident Report

Contains:

- Incident timeline
- Impact assessment
- Root cause analysis
- Recommendations

➡️ [Open Report](./Post-Incident-Report.md)

---

## Network Diagram

Visual representation of the project infrastructure.

➡️ [Open Diagram](./network-diagram.md)

---

# 🔍 Key Findings

- SIEM dashboards significantly improved visibility.
- Reconnaissance activity was detected successfully.
- Authentication anomalies were identified quickly.
- ELK Stack proved effective for centralized monitoring.
- Proper log enrichment enhanced investigation quality.

---

# 🛠 Skills Demonstrated

- Security Information and Event Management (SIEM)
- ELK Stack Administration
- Log Analysis
- Incident Response
- Threat Detection
- Dashboard Development
- Security Monitoring
- Linux Administration
- Network Reconnaissance Analysis
- Technical Documentation

---

# 📚 Lessons Learned

This capstone project provided practical exposure to the responsibilities of a SOC Analyst by demonstrating how security events can be collected, correlated, visualized, and investigated within a centralized monitoring platform.

The experience strengthened my understanding of:

- Building a SIEM from scratch.
- Handling real log ingestion challenges.
- Investigating suspicious activities.
- Presenting findings through dashboards.
- Producing professional incident documentation.

---

# 👨‍💻 Author

**Shubham Chaurasiya**

BCA Student | Cybersecurity Enthusiast | SOC Analyst Aspirant

GitHub: https://github.com/professorshubhx

LinkedIn: https://linkedin.com/in/shubham-chaurasiya-60932a359

---

## ⭐ Acknowledgements

This project was completed as part of the **ApexPlanet Cybersecurity Internship Program** and represents the culmination of hands-on learning in SIEM operations and incident response.

If you found this project useful, consider giving the repository a ⭐.
