# Day 27 — Kibana Dashboards and Visualizations
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## What We're Building Today

Today we build the actual SIEM dashboards in Kibana — the visual interface that a SOC analyst stares at all day. Four dashboards:

```
Dashboard 1: Security Overview        — top-level summary of all activity
Dashboard 2: Authentication Monitor   — SSH brute force, login tracking
Dashboard 3: Web Application Security — Apache attack detection
Dashboard 4: Network Activity         — Nmap scan results, port data
```

Each dashboard has multiple visualizations — bar charts, pie charts, data tables, metric counters, and time-series line charts.

---

## Kibana Visualization Types

```
Metric         Single number — "Total failed logins: 1,247"
Bar chart      Compare values across categories
Pie/Donut      Show proportions — "60% SQLi, 30% Scanner, 10% LFI"
Line chart     Show trends over time
Area chart     Like line chart but filled — good for volume
Data table     Tabular view — top attacking IPs with counts
Tag cloud      Word cloud from field values
Heat map       2D intensity map — attacks by hour and day
TSVB           Time Series Visual Builder — advanced time charts
```

---

## How to Create a Visualization

```
1. Open http://localhost:5601
2. Hamburger menu (≡) → Visualize Library
3. Click "Create visualization"
4. Choose visualization type
5. Select index pattern (logs-auth-*, logs-apache-*, etc.)
6. Configure buckets and metrics
7. Click Save → give it a name
8. Add to a dashboard later
```

---

## Dashboard 1 — Security Overview

This is the first dashboard a SOC analyst sees — a high-level summary of everything happening right now.

### Visualization 1.1 — Total Events (Metric)

```
Type: Metric
Index: logs-*
Metric: Count
Title: Total Security Events

Steps:
1. Visualize Library → Create → Metric
2. Index: logs-*
3. Metric: Count (default)
4. Save as: "Total Security Events"
```

### Visualization 1.2 — Events by Severity (Pie Chart)

```
Type: Pie
Index: logs-*
Title: Events by Severity

Steps:
1. Create → Pie
2. Index: logs-*
3. Slices bucket:
   Aggregation: Terms
   Field: severity.keyword
   Size: 5
4. Save as: "Events by Severity"

Expected slices: high, medium, low, info
```

### Visualization 1.3 — Top Source IPs (Bar Chart)

```
Type: Horizontal Bar
Index: logs-*
Title: Top 10 Attacking IPs

Steps:
1. Create → Horizontal Bar
2. Index: logs-*
3. Y-axis: Count
4. X-axis bucket:
   Aggregation: Terms
   Field: src_ip.keyword (or clientip.keyword for Apache)
   Size: 10
5. Save as: "Top Attacking IPs"
```

### Visualization 1.4 — Events Over Time (Line Chart)

```
Type: Line
Index: logs-*
Title: Security Events Over Time

Steps:
1. Create → Line
2. Index: logs-*
3. Y-axis: Count
4. X-axis bucket:
   Aggregation: Date Histogram
   Field: @timestamp
   Interval: Auto
5. Save as: "Events Over Time"
```

### Visualization 1.5 — Events by Type (Data Table)

```
Type: Data Table
Index: logs-*
Title: Event Types Summary

Steps:
1. Create → Data Table
2. Index: logs-*
3. Metric: Count
4. Split rows bucket:
   Aggregation: Terms
   Field: event_type.keyword
   Size: 10
5. Save as: "Event Types Summary"
```

---

## Dashboard 2 — Authentication Monitor

### Visualization 2.1 — Failed vs Successful Logins (Bar)

```
Type: Vertical Bar
Index: logs-auth-*
Title: SSH Login Attempts

Steps:
1. Create → Vertical Bar
2. Index: logs-auth-*
3. Y-axis: Count
4. X-axis: Date Histogram → @timestamp → Auto
5. Split series:
   Sub-aggregation: Terms
   Field: event_type.keyword
6. Save as: "SSH Login Attempts Over Time"

Result: Two colored bars — failed vs successful logins
```

### Visualization 2.2 — Top Attacking IPs (Data Table)

```
Type: Data Table
Index: logs-auth-*
Title: Brute Force Source IPs

Steps:
1. Create → Data Table
2. Index: logs-auth-*
3. Filter: event_type.keyword is ssh_failed_login
4. Metric: Count
5. Split rows:
   Aggregation: Terms
   Field: src_ip.keyword
   Size: 10
   Order: Descending by Count
6. Save as: "Top Brute Force IPs"

Result: Table showing which IPs have most failed logins
```

### Visualization 2.3 — Failed Login Count (Metric)

```
Type: Metric
Index: logs-auth-*
Title: Total Failed SSH Logins

Steps:
1. Create → Metric
2. Index: logs-auth-*
3. Add filter: event_type.keyword is ssh_failed_login
4. Metric: Count
5. Save as: "Failed Login Count"
```

### Visualization 2.4 — Targeted Usernames (Pie)

```
Type: Pie
Index: logs-auth-*
Title: Most Targeted Usernames

Steps:
1. Create → Pie
2. Index: logs-auth-*
3. Filter: event_type is ssh_failed_login
4. Slices: Terms → failed_user.keyword → Size 10
5. Save as: "Targeted Usernames"
```

---

## Dashboard 3 — Web Application Security

### Visualization 3.1 — Attack Types Distribution (Pie)

```
Type: Donut (Pie variant)
Index: logs-apache-*
Title: Web Attack Types

Steps:
1. Create → Pie
2. Index: logs-apache-*
3. Add filter: attack_type exists
4. Slices: Terms → attack_type.keyword → Size 10
5. Save as: "Web Attack Distribution"

Expected slices:
  possible_sqli_xss
  path_traversal
  scanner_detected
```

### Visualization 3.2 — HTTP Response Codes (Bar)

```
Type: Vertical Bar
Index: logs-apache-*
Title: HTTP Response Codes

Steps:
1. Create → Vertical Bar
2. Index: logs-apache-*
3. Y-axis: Count
4. X-axis: Terms → response (integer field) → Size 10
5. Save as: "HTTP Response Codes"

Expected bars: 200, 301, 302, 403, 404, 500
```

### Visualization 3.3 — Top Attacked URLs (Data Table)

```
Type: Data Table
Index: logs-apache-*
Title: Most Targeted URLs

Steps:
1. Create → Data Table
2. Index: logs-apache-*
3. Filter: attack_type exists
4. Metric: Count
5. Split rows: Terms → request.keyword → Size 10
6. Save as: "Top Attacked URLs"
```

### Visualization 3.4 — Web Traffic Over Time (Area)

```
Type: Area
Index: logs-apache-*
Title: Web Traffic Over Time

Steps:
1. Create → Area
2. Index: logs-apache-*
3. Y-axis: Count
4. X-axis: Date Histogram → @timestamp → Auto
5. Split series: Terms → response (shows traffic by status code)
6. Save as: "Web Traffic Over Time"
```

### Visualization 3.5 — Scanner Activity (Metric)

```
Type: Metric
Index: logs-apache-*
Title: Scanner Detections

Steps:
1. Create → Metric
2. Index: logs-apache-*
3. Filter: attack_type.keyword is scanner_detected
4. Metric: Count
5. Save as: "Scanner Detections"
```

---

## Dashboard 4 — Network Activity

### Visualization 4.1 — Open Ports Discovered (Bar)

```
Type: Horizontal Bar
Index: logs-nmap-*
Title: Open Ports on Target

Steps:
1. Create → Horizontal Bar
2. Index: logs-nmap-*
3. Y-axis: Count
4. X-axis: Terms → port.keyword → Size 30
5. Save as: "Open Ports Discovered"
```

### Visualization 4.2 — Services Detected (Tag Cloud)

```
Type: Tag Cloud
Index: logs-nmap-*
Title: Services Detected

Steps:
1. Create → Tag Cloud
2. Index: logs-nmap-*
3. Tags: Terms → service.keyword → Size 20
4. Save as: "Services Tag Cloud"
```

---

## Creating the Dashboards

After creating all visualizations, assemble them into dashboards:

```
1. Hamburger menu → Dashboard
2. Click "Create dashboard"
3. Click "Add from library"
4. Select your saved visualizations
5. Drag and resize panels
6. Save dashboard with a name
```

### Dashboard Layout Suggestion

```
Security Overview Dashboard:
┌─────────────────┬─────────────────┬──────────────────┐
│  Total Events   │  Failed Logins  │  Scanner Detect  │
│   (Metric)      │   (Metric)      │    (Metric)       │
├─────────────────┴─────────────────┴──────────────────┤
│              Events Over Time (Line Chart)            │
├───────────────────────────┬───────────────────────────┤
│   Events by Severity      │    Top Attacking IPs      │
│      (Pie Chart)          │    (Bar Chart)            │
├───────────────────────────┴───────────────────────────┤
│              Event Types Summary (Data Table)         │
└───────────────────────────────────────────────────────┘
```

---

## Using Kibana Lens (Modern Alternative)

Kibana Lens is the newer, drag-and-drop visualization builder — easier than the classic editor:

```
1. Hamburger → Visualize Library → Create visualization
2. Choose "Lens" (recommended)
3. Select index pattern
4. Drag fields from left panel to chart area
5. Kibana suggests chart types automatically
6. Change chart type from dropdown
```

Lens example — creating failed login count by IP:

```
1. Open Lens
2. Index: logs-auth-*
3. Drag "src_ip.keyword" to X-axis
4. Drag "Count" to Y-axis
5. Change type to "Bar"
6. Add filter: event_type: ssh_failed_login
7. Save
```

---

## Kibana Alerts (Detection Rules)

Set up basic alerts that fire when attack thresholds are exceeded:

```
1. Hamburger → Stack Management → Rules
2. Click "Create rule"
3. Rule type: Elasticsearch query

Rule 1 — Brute Force Detection:
  Name: SSH Brute Force Alert
  Index: logs-auth-*
  Query: event_type: "ssh_failed_login"
  Threshold: count > 10
  Time window: 1 minute
  Action: Write to log

Rule 2 — SQLi Detection:
  Name: SQL Injection Alert
  Index: logs-apache-*
  Query: attack_type: "possible_sqli_xss"
  Threshold: count > 1
  Time window: 5 minutes
  Action: Write to log
```

---

## Taking Dashboard Screenshots

For your Capstone Report, take screenshots of:

```
1. Security Overview dashboard — full view
2. Authentication Monitor — showing brute force data
3. Top attacking IPs table — with real IP addresses
4. Web Attack Distribution pie — showing attack types
5. Events over time line chart — showing attack spike
6. Kibana Discover — showing parsed log events with fields
```

In Kibana, each panel has a camera icon for taking panel screenshots, or use the browser's full-page screenshot.

---

## Practice Tasks

1. Create the "Total Security Events" metric visualization
2. Create the "Events by Severity" pie chart
3. Create the "Events Over Time" line chart
4. Create "SSH Login Attempts Over Time" bar chart with split series
5. Create "Top Brute Force IPs" data table
6. Create "Web Attack Distribution" donut chart
7. Create "HTTP Response Codes" bar chart
8. Assemble Dashboard 1 (Security Overview) with at least 4 panels
9. Assemble Dashboard 2 (Authentication Monitor) with at least 3 panels
10. Take screenshots of both dashboards — save to `screenshots/` folder

---

## Quick Reference

```
VISUALIZATION TYPES             GOOD FOR
Metric         Single number   KPI counts
Bar (vertical) Time trends     Events over time
Bar (horiz.)   Category rank   Top IPs, top URLs
Pie/Donut      Proportions     Attack type split
Data Table     Detailed view   IP tables, URL tables
Line           Trends          Traffic over time
Area           Volume          Filled trend chart
Tag Cloud      Text frequency  Services, usernames

KQL FILTERS IN VISUALIZATIONS
event_type: "ssh_failed_login"
attack_type: *
severity: "high"
src_ip: "192.168.56.101"
response >= 400
```

---

## Resources

- Kibana Dashboard Guide: https://www.elastic.co/guide/en/kibana/current/dashboard.html
- Kibana Lens: https://www.elastic.co/guide/en/kibana/current/lens.html
- Kibana Alerting: https://www.elastic.co/guide/en/kibana/current/alerting-getting-started.html
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 27 Complete | Next: Day 28 — Incident Response Simulation*
