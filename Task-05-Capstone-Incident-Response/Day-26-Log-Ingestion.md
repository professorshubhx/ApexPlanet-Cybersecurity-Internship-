# Day 26 — Log Ingestion and Index Management
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## What We're Doing Today

Yesterday we installed the ELK Stack. Today we get logs into Elasticsearch. This file is updated specifically for **Kali Linux** — commands are tested and working on Kali.

**Important note about Kali Linux:**
- Kali does NOT have `/var/log/auth.log` by default — needs rsyslog setup
- Apache logs are on Kali itself (localhost), not Metasploitable2
- We use Kali's own Apache for web attack log generation

---

## Log Flow Architecture

```
Kali Linux
├── /var/log/auth.log      ──► Logstash auth-log.conf  ──► logs-auth-*
├── /var/log/apache2/      ──► Logstash apache-log.conf──► logs-apache-*
└── /root/nmap-*.xml       ──► Logstash nmap-log.conf  ──► logs-nmap-*
                                        │
                                        ▼
                               Elasticsearch :9200
                                        │
                                        ▼
                                  Kibana :5601
```

---

## Step 1 — Fix auth.log on Kali

Kali Linux uses `journald` by default — `auth.log` does not exist until rsyslog is configured.

```bash
# Step 1a: Install rsyslog
sudo apt install rsyslog -y

# Step 1b: Enable auth logging
sudo bash -c 'echo "auth,authpriv.*    /var/log/auth.log" > /etc/rsyslog.d/auth.conf'

# Step 1c: Start rsyslog
sudo systemctl enable rsyslog
sudo systemctl restart rsyslog

# Step 1d: Create the file with correct permissions
sudo touch /var/log/auth.log
sudo chmod 640 /var/log/auth.log
sudo chown root:adm /var/log/auth.log

# Step 1e: Verify file exists
ls -la /var/log/auth.log
```

---

## Step 2 — Generate auth.log Entries

```bash
# Method A: SSH failed attempts to localhost
for i in {1..15}; do
  ssh wronguser@127.0.0.1 \
    -o StrictHostKeyChecking=no \
    -o BatchMode=yes \
    -o ConnectTimeout=3 2>/dev/null
  sleep 0.5
done

# Method B: SSH failed attempts to Metasploitable2
# (these show in Kali's auth.log as outbound, and Metasploitable's as inbound)
for i in {1..10}; do
  ssh wronguser@192.168.56.104 \
    -o StrictHostKeyChecking=no \
    -o BatchMode=yes \
    -o ConnectTimeout=3 2>/dev/null
  sleep 0.5
done

# Method C: Direct inject (fastest, always works)
sudo bash -c 'cat >> /var/log/auth.log << EOF
Jun  7 10:00:01 kali sshd[1001]: Failed password for msfadmin from 192.168.56.101 port 45001 ssh2
Jun  7 10:00:02 kali sshd[1002]: Failed password for root from 192.168.56.101 port 45002 ssh2
Jun  7 10:00:03 kali sshd[1003]: Failed password for admin from 192.168.56.101 port 45003 ssh2
Jun  7 10:00:04 kali sshd[1004]: Failed password for msfadmin from 192.168.56.101 port 45004 ssh2
Jun  7 10:00:05 kali sshd[1005]: Failed password for user from 192.168.56.101 port 45005 ssh2
Jun  7 10:00:06 kali sshd[1006]: Accepted password for msfadmin from 192.168.56.101 port 45006 ssh2
Jun  7 10:00:07 kali sshd[1007]: Failed password for msfadmin from 192.168.56.101 port 45007 ssh2
Jun  7 10:00:08 kali sshd[1008]: Failed password for root from 192.168.56.101 port 45008 ssh2
Jun  7 10:00:09 kali sshd[1009]: Failed password for admin from 192.168.56.101 port 45009 ssh2
Jun  7 10:00:10 kali sshd[1010]: Failed password for msfadmin from 192.168.56.101 port 45010 ssh2
EOF'

# Verify entries exist
sudo tail -10 /var/log/auth.log
```

---

## Step 3 — Fix Apache on Kali and Generate Web Logs

Web attack logs come from **Kali's own Apache** — not Metasploitable2.

```bash
# Step 3a: Make sure Apache is running on Kali
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2

# Step 3b: Verify Apache is running
sudo systemctl status apache2 --no-pager
curl -s http://localhost/ | head -3

# Step 3c: Check log file exists
ls -la /var/log/apache2/

# Step 3d: Generate normal web traffic to Kali's Apache
curl http://localhost/
curl http://localhost/index.html
curl http://localhost/nonexistent-page

# Step 3e: Generate SQL injection attack logs
curl "http://localhost/?id=1+UNION+SELECT+user(),database()--"
curl "http://localhost/?id=1+OR+1=1--"
curl "http://localhost/?search=SELECT+*+FROM+users"
curl "http://localhost/?id=1'+ORDER+BY+2--+-"
curl "http://localhost/?input=DROP+TABLE+users"

# Step 3f: Generate XSS attack logs
curl "http://localhost/?name=<script>alert(1)</script>"
curl "http://localhost/?q=<img+src=x+onerror=alert(1)>"

# Step 3g: Generate path traversal logs
curl "http://localhost/?page=../../etc/passwd"
curl "http://localhost/?file=../../../etc/shadow"
curl "http://localhost/../../../../etc/passwd"

# Step 3h: Generate scanner simulation logs
curl -A "Nikto/2.1.6" http://localhost/
curl -A "sqlmap/1.7.0" http://localhost/
curl -A "Nmap Scripting Engine" http://localhost/
curl -A "masscan/1.3" http://localhost/

# Step 3i: Verify Apache logs
sudo tail -20 /var/log/apache2/access.log
```

---

## Step 4 — Fix Logstash Config for Kali

Update auth-log.conf to work correctly on Kali:

```bash
sudo nano /etc/logstash/conf.d/auth-log.conf
```

Replace entire content with:

```ruby
input {
  file {
    path => "/var/log/auth.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "auth"
    tags => ["auth", "ssh"]
  }
}

filter {
  if [type] == "auth" {
    grok {
      match => {
        "message" => "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:hostname} %{WORD:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:log_message}"
      }
      overwrite => ["message"]
    }

    if "Failed password" in [log_message] {
      grok {
        match => {
          "log_message" => "Failed password for (?:invalid user )?%{USERNAME:failed_user} from %{IP:src_ip} port %{INT:src_port}"
        }
      }
      mutate {
        add_field => { "event_type" => "ssh_failed_login" }
        add_field => { "severity" => "high" }
      }
    }

    if "Accepted" in [log_message] {
      grok {
        match => {
          "log_message" => "Accepted %{WORD:auth_method} for %{USERNAME:success_user} from %{IP:src_ip} port %{INT:src_port}"
        }
      }
      mutate {
        add_field => { "event_type" => "ssh_successful_login" }
        add_field => { "severity" => "info" }
      }
    }

    date {
      match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      target => "@timestamp"
    }
  }
}

output {
  if [type] == "auth" {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      index => "logs-auth-%{+YYYY.MM.dd}"
    }
  }
}
```

Update apache-log.conf:

```bash
sudo nano /etc/logstash/conf.d/apache-log.conf
```

Replace with:

```ruby
input {
  file {
    path => "/var/log/apache2/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "apache"
    tags => ["apache", "web"]
  }
}

filter {
  if [type] == "apache" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }

    mutate {
      convert => {
        "response" => "integer"
        "bytes" => "integer"
      }
    }

    if [request] =~ /(?i)(union|select|insert|drop|delete|update|script|alert)/ {
      mutate {
        add_field => { "attack_type" => "possible_sqli_xss" }
        add_field => { "severity" => "high" }
        add_tag => ["attack"]
      }
    }

    if [request] =~ /\.\.\// {
      mutate {
        add_field => { "attack_type" => "path_traversal" }
        add_field => { "severity" => "high" }
        add_tag => ["attack"]
      }
    }

    if [agent] =~ /(?i)(nikto|sqlmap|nmap|masscan|dirbuster|gobuster)/ {
      mutate {
        add_field => { "attack_type" => "scanner_detected" }
        add_field => { "severity" => "medium" }
        add_tag => ["scanner"]
      }
    }

    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
      target => "@timestamp"
    }
  }
}

output {
  if [type] == "apache" {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      index => "logs-apache-%{+YYYY.MM.dd}"
    }
  }
}
```

---

## Step 5 — Fix Filebeat Config for Kali

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Replace entire content with this clean config:

```yaml
filebeat.inputs:

- type: log
  enabled: true
  paths:
    - /var/log/auth.log
  tags: ["auth", "syslog"]
  fields:
    log_type: auth

- type: log
  enabled: true
  paths:
    - /var/log/apache2/access.log
  tags: ["apache", "web"]
  fields:
    log_type: apache

output.logstash:
  hosts: ["localhost:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

---

## Step 6 — Restart All Services in Correct Order

```bash
# Always restart in this order
echo "Restarting Elasticsearch..."
sudo systemctl restart elasticsearch
sleep 30

echo "Restarting Logstash..."
sudo systemctl restart logstash
sleep 45

echo "Restarting Filebeat..."
sudo systemctl restart filebeat
sleep 10

echo "Restarting Kibana..."
sudo systemctl restart kibana
sleep 30

# Verify all running
for svc in elasticsearch logstash kibana filebeat; do
  echo -n "$svc: "
  sudo systemctl is-active $svc
done
```

---

## Step 7 — Verify Elasticsearch Has Data

```bash
# Wait 2 minutes after restart for logs to process
sleep 120

# Check indices
curl -s "http://localhost:9200/_cat/indices?v" | grep -E "logs-|health"

# Count auth documents
curl -s "http://localhost:9200/logs-auth-*/_count" 2>/dev/null | python3 -m json.tool

# Count apache documents
curl -s "http://localhost:9200/logs-apache-*/_count" 2>/dev/null | python3 -m json.tool
```

---

## Step 8 — If Logstash Still Not Working — Direct Push Method

This method bypasses Logstash completely and pushes directly to Elasticsearch. **Use this if Logstash pipelines are not working.**

```bash
# Push auth log events directly to Elasticsearch
TODAY=$(date +%Y.%m.%d)

for i in $(seq 1 20); do
  curl -s -X POST "http://localhost:9200/logs-auth-${TODAY}/_doc" \
    -H 'Content-Type: application/json' \
    -d "{
      \"@timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"event_type\": \"ssh_failed_login\",
      \"src_ip\": \"192.168.56.101\",
      \"failed_user\": \"msfadmin\",
      \"severity\": \"high\",
      \"hostname\": \"kali\",
      \"program\": \"sshd\",
      \"message\": \"Failed password for msfadmin from 192.168.56.101 port 4500${i} ssh2\"
    }" > /dev/null
  sleep 0.1
done

# Push one successful login
curl -s -X POST "http://localhost:9200/logs-auth-${TODAY}/_doc" \
  -H 'Content-Type: application/json' \
  -d "{
    \"@timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
    \"event_type\": \"ssh_successful_login\",
    \"src_ip\": \"192.168.56.101\",
    \"success_user\": \"msfadmin\",
    \"severity\": \"info\",
    \"hostname\": \"kali\",
    \"message\": \"Accepted password for msfadmin from 192.168.56.101 port 45021 ssh2\"
  }" > /dev/null

# Push Apache attack events
for attack in "possible_sqli_xss" "possible_sqli_xss" "path_traversal" "scanner_detected" "possible_sqli_xss"; do
  curl -s -X POST "http://localhost:9200/logs-apache-${TODAY}/_doc" \
    -H 'Content-Type: application/json' \
    -d "{
      \"@timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"clientip\": \"192.168.56.101\",
      \"attack_type\": \"${attack}\",
      \"severity\": \"high\",
      \"verb\": \"GET\",
      \"request\": \"/dvwa/?id=1+UNION+SELECT+user(),database()\",
      \"response\": 200,
      \"bytes\": 1234,
      \"agent\": \"Mozilla/5.0\"
    }" > /dev/null
  sleep 0.1
done

echo "Data pushed. Verifying..."
sleep 5
curl -s "http://localhost:9200/_cat/indices?v" | grep logs-
```

---

## Step 9 — Verify in Kibana

```
1. Open http://localhost:5601
2. Hamburger menu → Stack Management → Index Patterns
3. Create index pattern: logs-auth-*  (timestamp: @timestamp)
4. Create index pattern: logs-apache-* (timestamp: @timestamp)
5. Hamburger menu → Discover
6. Select logs-auth-* → set time to Last 7 days
7. You should see events
```

KQL queries to test:

```
event_type : "ssh_failed_login"
severity : "high"
attack_type : *
src_ip : "192.168.56.101"
```

---

## Complete Verification Checklist

```bash
# Run all these — tick each one that works
echo "=== 1. Services ==="
for svc in elasticsearch logstash kibana filebeat; do
  echo -n "  $svc: "; sudo systemctl is-active $svc
done

echo "=== 2. Auth Log ==="
sudo wc -l /var/log/auth.log

echo "=== 3. Apache Log ==="
sudo wc -l /var/log/apache2/access.log

echo "=== 4. Elasticsearch Indices ==="
curl -s "http://localhost:9200/_cat/indices?v" | grep logs-

echo "=== 5. Auth Document Count ==="
curl -s "http://localhost:9200/logs-auth-*/_count" 2>/dev/null

echo "=== 6. Apache Document Count ==="
curl -s "http://localhost:9200/logs-apache-*/_count" 2>/dev/null
```

All 6 should show data. If indices show 0 or missing — use Step 8 Direct Push method.

---

## Troubleshooting — Kali Specific

| Problem | Cause | Fix |
|---------|-------|-----|
| `/var/log/auth.log` missing | rsyslog not configured | Step 1 above |
| auth.log empty after setup | No SSH attempts made | Step 2 Method C (inject) |
| Apache log missing | Apache not started | `sudo systemctl start apache2` |
| Logstash no index created | Pipeline config error | Use Step 8 direct push |
| Kibana no data | Wrong time range | Set to Last 7 days in Discover |
| Filebeat not connecting | Logstash port 5044 not open | Check `sudo ss -tuln \| grep 5044` |

---

## Quick Reference

```
KALI LOG LOCATIONS              GENERATE LOGS
/var/log/auth.log               Method A: ssh wronguser@127.0.0.1
/var/log/apache2/access.log     Method B: inject directly (Step 2C)
/var/log/syslog                 Method C: curl attack URLs (Step 3)
/root/nmap-*.xml                Method D: direct ES push (Step 8)

RESTART ORDER                   VERIFY COMMANDS
1. elasticsearch (sleep 30)     curl localhost:9200/_cat/indices?v
2. logstash (sleep 45)          curl localhost:9200/logs-auth-*/_count
3. filebeat (sleep 10)          sudo tail -10 /var/log/auth.log
4. kibana (sleep 30)            sudo tail -10 /var/log/apache2/access.log
```

---

## Resources

- Kibana Query Language: https://www.elastic.co/guide/en/kibana/current/kuery-query.html
- Grok Debugger: https://grokdebug.herokuapp.com/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 26 Complete | Next: Day 27 — Kibana Dashboards and Visualizations*
