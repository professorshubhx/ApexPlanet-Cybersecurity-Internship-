# Day 26 — Log Ingestion and Index Management
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## What We're Doing Today

Yesterday we installed the ELK Stack. Today we focus on actually getting logs into Elasticsearch — configuring Logstash pipelines properly, generating real attack traffic to create meaningful log data, verifying data is flowing, and setting up index patterns in Kibana so we can search and visualize.

By the end of today, Kibana's Discover tab should show real log events from our lab.

---

## Log Ingestion Architecture Recap

```
Attack Activity          Log Files              Logstash          Elasticsearch
───────────────          ─────────              ────────          ─────────────
Hydra SSH brute ──► /var/log/auth.log ──► auth-log.conf  ──► logs-auth-*
Apache web reqs ──► /var/log/apache2/ ──► apache-log.conf──► logs-apache-*
Nmap scan       ──► nmap-output.xml   ──► nmap-log.conf  ──► logs-nmap-*
System events   ──► /var/log/syslog   ──► beats-input    ──► logs-system-*
```

---

## Step 1 — Verify All Services Running

```bash
# Check all four services
sudo systemctl status elasticsearch --no-pager
sudo systemctl status logstash --no-pager
sudo systemctl status kibana --no-pager
sudo systemctl status filebeat --no-pager

# Quick one-liner status check
for svc in elasticsearch logstash kibana filebeat; do
  echo -n "$svc: "
  sudo systemctl is-active $svc
done

# Verify Elasticsearch is responding
curl -s http://localhost:9200 | python3 -m json.tool
```

---

## Step 2 — Generate Real Attack Log Data

We need actual log entries before we can visualize anything. Run these attacks to generate meaningful data:

### Generate SSH Brute Force Logs (auth.log)

```bash
# Run Hydra against Metasploitable2 SSH
# This generates MANY failed login entries in auth.log
hydra -l msfadmin \
      -P /usr/share/wordlists/rockyou.txt \
      ssh://192.168.56.104 \
      -t 4 -V \
      -o /tmp/hydra-results.txt &

# Let it run for 2-3 minutes then stop it
# Ctrl+C to stop

# Check auth.log has entries
sudo tail -30 /var/log/auth.log | grep "Failed\|Accepted"
```

### Generate Web Attack Logs (apache2/access.log)

```bash
# Make sure DVWA Apache is running on Metasploitable2 first
# Then generate various attack patterns:

# Normal traffic
curl -s http://192.168.56.104/dvwa/ > /dev/null

# SQL Injection attempts
curl -s "http://192.168.56.104/dvwa/vulnerabilities/sqli/?id=1'+UNION+SELECT+user(),database()--+-&Submit=Submit" > /dev/null
curl -s "http://192.168.56.104/dvwa/vulnerabilities/sqli/?id=1+OR+1=1--&Submit=Submit" > /dev/null
curl -s "http://192.168.56.104/dvwa/vulnerabilities/sqli/?id=1'+ORDER+BY+2--+-&Submit=Submit" > /dev/null

# XSS attempts
curl -s "http://192.168.56.104/dvwa/vulnerabilities/xss_r/?name=<script>alert(1)</script>" > /dev/null

# Path traversal / LFI
curl -s "http://192.168.56.104/dvwa/vulnerabilities/fi/?page=../../../../../../etc/passwd" > /dev/null

# Scanner simulation
curl -s -A "Nikto/2.1.6" http://192.168.56.104/ > /dev/null
curl -s -A "sqlmap/1.7" http://192.168.56.104/dvwa/ > /dev/null

# Brute force login attempts
for i in {1..20}; do
  curl -s -X POST http://192.168.56.104/dvwa/login.php \
    -d "username=admin&password=wrongpass$i&Login=Login" > /dev/null
done

# Check apache logs have entries
sudo tail -20 /var/log/apache2/access.log
```

### Generate Nmap Scan Data

```bash
# Run Nmap and save XML output (Logstash reads this)
sudo nmap -sV -sC 192.168.56.104 \
  -oX /root/nmap-lab-scan.xml \
  -oN /root/nmap-lab-scan.txt

# Verify XML was created
ls -lh /root/nmap-lab-scan.xml
cat /root/nmap-lab-scan.xml | head -20
```

---

## Step 3 — Verify Logstash is Processing

```bash
# Watch Logstash logs in real time
sudo tail -f /var/log/logstash/logstash-plain.log

# Look for lines like:
# [INFO] Pipelines running: auth, apache
# [INFO] Successfully started Logstash API endpoint

# Test auth pipeline manually
sudo /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  -f /etc/logstash/conf.d/auth-log.conf
# Should output: Configuration OK

# Test apache pipeline
sudo /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  -f /etc/logstash/conf.d/apache-log.conf
# Should output: Configuration OK
```

---

## Step 4 — Verify Data in Elasticsearch

```bash
# List all indices — you should see logs-auth-*, logs-apache-*
curl -s "http://localhost:9200/_cat/indices?v&s=index"

# Count documents in auth index
curl -s "http://localhost:9200/logs-auth-*/_count" | python3 -m json.tool

# Count documents in apache index
curl -s "http://localhost:9200/logs-apache-*/_count" | python3 -m json.tool

# Sample a document from auth index
curl -s "http://localhost:9200/logs-auth-*/_search?size=1&pretty"

# Search for failed SSH logins specifically
curl -s -X GET "http://localhost:9200/logs-auth-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match": {
        "event_type": "ssh_failed_login"
      }
    },
    "size": 5
  }'

# Search for attack events in Apache logs
curl -s -X GET "http://localhost:9200/logs-apache-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "exists": {
        "field": "attack_type"
      }
    },
    "size": 5
  }'
```

---

## Step 5 — Create Index Patterns in Kibana

Index patterns tell Kibana which Elasticsearch indices to query.

### Via Kibana UI

```
1. Open browser: http://localhost:5601
2. Click hamburger menu (≡) → Stack Management
3. Click "Index Patterns" (under Kibana section)
4. Click "Create index pattern"

Create these three patterns:

Pattern 1: logs-auth-*
  - Name: logs-auth-*
  - Timestamp field: @timestamp
  - Click Create

Pattern 2: logs-apache-*
  - Name: logs-apache-*
  - Timestamp field: @timestamp
  - Click Create

Pattern 3: logs-nmap-*
  - Name: logs-nmap-*
  - Timestamp field: @timestamp
  - Click Create

Pattern 4: logs-* (catch-all)
  - Name: logs-*
  - Timestamp field: @timestamp
  - Click Create
```

### Via API (faster)

```bash
# Create index pattern for auth logs
curl -s -X POST "http://localhost:5601/api/saved_objects/index-pattern/logs-auth" \
  -H 'kbn-xsrf: true' \
  -H 'Content-Type: application/json' \
  -d '{
    "attributes": {
      "title": "logs-auth-*",
      "timeFieldName": "@timestamp"
    }
  }'

# Create index pattern for apache logs
curl -s -X POST "http://localhost:5601/api/saved_objects/index-pattern/logs-apache" \
  -H 'kbn-xsrf: true' \
  -H 'Content-Type: application/json' \
  -d '{
    "attributes": {
      "title": "logs-apache-*",
      "timeFieldName": "@timestamp"
    }
  }'
```

---

## Step 6 — Explore Data in Kibana Discover

```
1. Open http://localhost:5601
2. Click hamburger menu → Discover
3. Select index pattern: logs-auth-*
4. Set time range: Last 24 hours (top right)
5. You should see SSH log events

Key fields to look for in auth logs:
  event_type    ssh_failed_login or ssh_successful_login
  src_ip        Source IP of login attempt
  failed_user   Username that was tried
  severity      medium or high
  hostname      Target machine name

6. Switch to logs-apache-*
Key fields:
  clientip      Source IP of web request
  request       URL requested
  response      HTTP response code
  attack_type   possible_sqli_xss / path_traversal / scanner_detected
  severity      high / medium
  agent         User-Agent string
```

### Using KQL (Kibana Query Language) in Discover

```
# Find all failed SSH logins
event_type : "ssh_failed_login"

# Find attacks from specific IP
src_ip : "192.168.56.101"

# Find all detected attacks in Apache
attack_type : *

# Find SQL injection attempts
request : *UNION* or request : *SELECT*

# Find scanner activity
attack_type : "scanner_detected"

# Find high severity events
severity : "high"

# Find events in time range (use UI time picker)
# Or: @timestamp >= "2026-06-01" and @timestamp <= "2026-06-03"
```

---

## Step 7 — Useful Elasticsearch API Queries

```bash
# Get field mapping for an index (see all available fields)
curl -s "http://localhost:9200/logs-auth-*/_mapping?pretty" | head -60

# Get unique values of a field (top source IPs)
curl -s -X GET "http://localhost:9200/logs-auth-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "aggs": {
      "top_source_ips": {
        "terms": {
          "field": "src_ip.keyword",
          "size": 10
        }
      }
    }
  }'

# Count failed logins per source IP
curl -s -X GET "http://localhost:9200/logs-auth-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "query": {
      "match": { "event_type": "ssh_failed_login" }
    },
    "aggs": {
      "attacks_by_ip": {
        "terms": {
          "field": "src_ip.keyword",
          "size": 10
        }
      }
    }
  }'

# Count attack types in Apache logs
curl -s -X GET "http://localhost:9200/logs-apache-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "aggs": {
      "attack_types": {
        "terms": {
          "field": "attack_type.keyword",
          "size": 10
        }
      }
    }
  }'
```

---

## Step 8 — Index Lifecycle Management (Optional)

For a real SIEM, you'd configure index lifecycle management to automatically roll over, compress, and delete old indices:

```bash
# Create a simple ILM policy
curl -s -X PUT "http://localhost:9200/_ilm/policy/siem-policy" \
  -H 'Content-Type: application/json' \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_size": "1gb",
              "max_age": "7d"
            }
          }
        },
        "delete": {
          "min_age": "30d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

---

## Troubleshooting Log Ingestion

**No indices appearing in Elasticsearch:**

```bash
# Check Logstash pipeline is running
sudo systemctl status logstash

# Check for pipeline errors
sudo grep -i "error\|exception" /var/log/logstash/logstash-plain.log | tail -20

# Restart Logstash
sudo systemctl restart logstash
sleep 30
curl "http://localhost:9200/_cat/indices?v"
```

**Auth log events not parsing correctly:**

```bash
# Test Grok pattern manually
# Go to: http://localhost:5601 → Dev Tools → Grok Debugger
# Or use online: https://grokdebug.herokuapp.com/

# Sample auth.log line to test:
# Jun  2 10:14:32 kali sshd[1234]: Failed password for msfadmin from 192.168.56.101 port 45678 ssh2

# Grok pattern to test:
# %{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:hostname} %{WORD:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:log_message}
```

**Apache logs not detecting attacks:**

```bash
# Check if attack patterns are in the logs
grep -i "union\|select\|passwd" /var/log/apache2/access.log | tail -10

# If empty — generate more attack traffic
curl "http://192.168.56.104/dvwa/vulnerabilities/sqli/?id=1'+UNION+SELECT+1,2--+-"
```

---

## What Good Data Looks Like

After completing today's steps, running this command:

```bash
curl -s "http://localhost:9200/_cat/indices?v&s=index"
```

Should show:

```
health  status  index                    docs.count  store.size
green   open    logs-apache-2026.06.02   847         2.1mb
green   open    logs-auth-2026.06.02     1234        1.8mb
green   open    logs-nmap-2026.06.02     26          512kb
```

And in Kibana Discover — you should see events with fields like `event_type`, `src_ip`, `attack_type` properly parsed.

---

## Practice Tasks

1. Run Hydra for 3 minutes against SSH — confirm entries in `/var/log/auth.log`
2. Run all the Apache attack curl commands — confirm entries in `/var/log/apache2/access.log`
3. Run `curl "http://localhost:9200/_cat/indices?v"` — confirm indices exist
4. Run the failed login count query — how many failed logins did Hydra generate?
5. Run the attack type aggregation — what attack types were detected?
6. Open Kibana Discover → `logs-auth-*` — find events with `event_type: ssh_failed_login`
7. In Kibana Discover → `logs-apache-*` — filter by `attack_type: *` — list all attack types found
8. In Kibana Discover — find the User-Agent `sqlmap` in Apache logs
9. Run `sudo nmap -sV 192.168.56.104 -oX /root/nmap-scan.xml` — verify XML created
10. Create all 4 index patterns in Kibana (logs-auth-*, logs-apache-*, logs-nmap-*, logs-*)

---

## Quick Reference

```
VERIFY DATA FLOW                KQL QUERIES IN KIBANA
curl localhost:9200/_cat/indices  event_type: "ssh_failed_login"
curl localhost:9200/logs-auth-*/_count  attack_type: *
                                severity: "high"
GENERATE TEST DATA              src_ip: "192.168.56.101"
hydra SSH (auth.log)            request: *UNION*
curl attack URLs (apache)       agent: *nikto*
nmap -oX (nmap index)

ELASTICSEARCH APIS              LOGSTASH CONFIG PATH
/_cat/indices?v                 /etc/logstash/conf.d/
/_search?pretty                 auth-log.conf
/_count                         apache-log.conf
/_mapping                       nmap-log.conf
/_cat/health                    beats-input.conf
```

---

## Resources

- Kibana Query Language: https://www.elastic.co/guide/en/kibana/current/kuery-query.html
- Elasticsearch Search API: https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html
- Grok Debugger: https://grokdebug.herokuapp.com/
- Logstash Grok Filter: https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 26 Complete | Next: Day 27 — Kibana Dashboards and Visualizations*
