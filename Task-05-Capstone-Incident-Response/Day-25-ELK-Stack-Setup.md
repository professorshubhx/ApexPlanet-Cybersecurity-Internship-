# Day 25 — ELK Stack Installation and Configuration
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## What We're Installing Today

The ELK Stack consists of three main components plus Filebeat:

```
Elasticsearch   The search and analytics engine — stores all logs
Logstash        Data processing pipeline — collects, parses, transforms logs
Kibana          Web interface — dashboards, search, visualization
Filebeat        Lightweight log shipper — sends logs to Logstash
```

All four need to be the same version. We'll install version 8.x.

**Time required:** 30–60 minutes depending on internet speed and RAM.

---

## Pre-Installation Checks

```bash
# Check available RAM (need at least 4GB)
free -h

# Check available disk space (need at least 10GB free)
df -h

# Check if Java is installed
java -version
# If not installed:
sudo apt install default-jdk -y
java -version
# Should show: openjdk 11.x or higher

# Update system first
sudo apt update && sudo apt upgrade -y
```

---

## Step 1 — Add Elastic Repository

```bash
# Install required dependencies
sudo apt install apt-transport-https curl gnupg -y

# Import Elastic GPG key
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Add Elastic repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Update package list
sudo apt update
```

---

## Step 2 — Install Elasticsearch

```bash
# Install Elasticsearch
sudo apt install elasticsearch -y

# Note: During installation, the output will show the auto-generated
# elastic user password — SAVE THIS PASSWORD
# It looks like: The generated password for the elastic built-in superuser is: xK9mP2qRvL3jT
```

### Configure Elasticsearch

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Find and modify/add these lines:

```yaml
# Cluster name
cluster.name: siem-lab

# Node name
node.name: kali-node-1

# Network host — allow connections from localhost only
network.host: 127.0.0.1

# HTTP port
http.port: 9200

# Disable security for lab use (easier setup)
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl:
  enabled: false
xpack.security.transport.ssl:
  enabled: false
```

### Start Elasticsearch

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# Wait 30 seconds then check status
sleep 30
sudo systemctl status elasticsearch

# Verify it's running
curl -X GET "http://localhost:9200"
```

Expected output:

```json
{
  "name" : "kali-node-1",
  "cluster_name" : "siem-lab",
  "version" : {
    "number" : "8.x.x"
  },
  "tagline" : "You Know, for Search"
}
```

---

## Step 3 — Install Kibana

```bash
# Install Kibana
sudo apt install kibana -y
```

### Configure Kibana

```bash
sudo nano /etc/kibana/kibana.yml
```

Modify these settings:

```yaml
# Kibana server port
server.port: 5601

# Kibana server host — accessible from browser
server.host: "0.0.0.0"

# Kibana server name
server.name: "siem-lab-kibana"

# Elasticsearch connection
elasticsearch.hosts: ["http://localhost:9200"]

# Disable security for lab
xpack.security.enabled: false
```

### Start Kibana

```bash
sudo systemctl enable kibana
sudo systemctl start kibana

# Check status
sudo systemctl status kibana
```

### Access Kibana in Browser

```
Open Kali browser and go to:
http://localhost:5601

Or from host machine:
http://192.168.56.101:5601
```

Kibana takes 1–2 minutes to fully load on first start. If you see "Kibana server is not ready yet" — wait a minute and refresh.

---

## Step 4 — Install Logstash

```bash
# Install Logstash
sudo apt install logstash -y
```

### Understanding Logstash Configuration

Logstash uses pipeline configuration files with three sections:

```
input   { }     Where to receive data FROM
filter  { }     How to parse and transform data
output  { }     Where to SEND processed data
```

### Create Logstash Pipeline — Auth Log

```bash
sudo nano /etc/logstash/conf.d/auth-log.conf
```

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

    # Parse the syslog format
    grok {
      match => {
        "message" => "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:hostname} %{WORD:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:log_message}"
      }
    }

    # Parse SSH failed password attempts
    if "Failed password" in [log_message] {
      grok {
        match => {
          "log_message" => "Failed password for (?:invalid user )?%{USERNAME:failed_user} from %{IP:src_ip} port %{INT:src_port}"
        }
      }
      mutate {
        add_field => { "event_type" => "ssh_failed_login" }
        add_field => { "severity" => "medium" }
      }
    }

    # Parse SSH successful login
    if "Accepted password" in [log_message] or "Accepted publickey" in [log_message] {
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

    # Parse date
    date {
      match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
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
  stdout {
    codec => rubydebug
  }
}
```

### Create Logstash Pipeline — Apache Access Log

```bash
sudo nano /etc/logstash/conf.d/apache-log.conf
```

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

    # Parse Apache Combined Log Format
    grok {
      match => {
        "message" => "%{COMBINEDAPACHELOG}"
      }
    }

    # Convert response code and bytes to integers
    mutate {
      convert => {
        "response" => "integer"
        "bytes" => "integer"
      }
    }

    # Detect SQL injection patterns
    if [request] =~ /(?i)(union|select|insert|drop|delete|update|exec|script|alert)/  {
      mutate {
        add_field => { "attack_type" => "possible_sqli_xss" }
        add_field => { "severity" => "high" }
        add_tag => ["attack"]
      }
    }

    # Detect path traversal
    if [request] =~ /\.\.\// {
      mutate {
        add_field => { "attack_type" => "path_traversal" }
        add_field => { "severity" => "high" }
        add_tag => ["attack"]
      }
    }

    # Detect scanner user agents
    if [agent] =~ /(?i)(nikto|sqlmap|nmap|masscan|dirbuster|gobuster)/ {
      mutate {
        add_field => { "attack_type" => "scanner_detected" }
        add_field => { "severity" => "medium" }
        add_tag => ["scanner"]
      }
    }

    # Parse date from Apache timestamp
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
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

### Create Logstash Pipeline — Nmap XML

```bash
sudo nano /etc/logstash/conf.d/nmap-log.conf
```

```ruby
input {
  file {
    path => "/root/nmap-*.xml"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "nmap"
    codec => "line"
    tags => ["nmap", "scan"]
  }
}

filter {
  if [type] == "nmap" {
    xml {
      source => "message"
      target => "nmap_data"
    }
    mutate {
      add_field => { "event_type" => "nmap_scan" }
    }
  }
}

output {
  if [type] == "nmap" {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      index => "logs-nmap-%{+YYYY.MM.dd}"
    }
  }
}
```

### Start Logstash

```bash
sudo systemctl enable logstash
sudo systemctl start logstash

# Check status
sudo systemctl status logstash

# Watch Logstash logs to see if pipelines load correctly
sudo tail -f /var/log/logstash/logstash-plain.log
```

---

## Step 5 — Install and Configure Filebeat

Filebeat is a lightweight agent that watches log files and ships new entries to Logstash.

```bash
# Install Filebeat
sudo apt install filebeat -y
```

### Configure Filebeat

```bash
sudo nano /etc/filebeat/filebeat.yml
```

```yaml
# ====== Filebeat inputs ======
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
    - /var/log/apache2/error.log
  tags: ["apache", "web"]
  fields:
    log_type: apache

- type: log
  enabled: true
  paths:
    - /var/log/syslog
  tags: ["syslog"]
  fields:
    log_type: syslog

# ====== Output to Logstash ======
output.logstash:
  hosts: ["localhost:5044"]

# ====== Disable Elasticsearch direct output ======
# output.elasticsearch:
#   hosts: ["localhost:9200"]

# ====== Logging ======
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

### Start Filebeat

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat

# Check status
sudo systemctl status filebeat

# Check Filebeat logs
sudo tail -f /var/log/filebeat/filebeat
```

---

## Step 6 — Add Filebeat Input to Logstash

Logstash needs to listen for Filebeat connections:

```bash
sudo nano /etc/logstash/conf.d/beats-input.conf
```

```ruby
input {
  beats {
    port => 5044
  }
}
```

Restart Logstash:

```bash
sudo systemctl restart logstash
```

---

## Verify the Entire Stack

```bash
# Check all services are running
sudo systemctl status elasticsearch
sudo systemctl status logstash
sudo systemctl status kibana
sudo systemctl status filebeat

# Check Elasticsearch indices (logs should appear here)
curl -X GET "http://localhost:9200/_cat/indices?v"

# You should see something like:
# health  index                    docs.count
# green   logs-auth-2026.06.02     1234
# green   logs-apache-2026.06.02   5678
```

### Access Kibana

```
Browser: http://localhost:5601

Navigate to:
  Menu → Discover → Select index pattern → logs-auth-*
  You should see your auth.log events streaming in
```

---

## Troubleshooting Common Issues

**Elasticsearch won't start — Out of Memory**

```bash
# Reduce Java heap size for low-RAM systems
sudo nano /etc/elasticsearch/jvm.options

# Change these lines:
-Xms512m      # Minimum heap (was 1g)
-Xmx512m      # Maximum heap (was 1g)

sudo systemctl restart elasticsearch
```

**Kibana shows "Kibana server is not ready yet"**

```bash
# Wait 2 minutes, then check
sudo systemctl status kibana
sudo journalctl -u kibana -f    # Watch live logs

# If elasticsearch connection fails:
curl http://localhost:9200       # Elasticsearch must be running first
```

**Logstash pipeline errors**

```bash
# Test pipeline config before starting
sudo /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  -f /etc/logstash/conf.d/auth-log.conf

# Should output: Configuration OK
```

**No data appearing in Kibana**

```bash
# Check indices exist
curl "http://localhost:9200/_cat/indices?v"

# Check Logstash received data
sudo tail -50 /var/log/logstash/logstash-plain.log

# Generate some log entries
sudo tail -f /var/log/auth.log    # Confirm logs exist
# Try SSH with wrong password to generate entries:
ssh wronguser@localhost            # Generates failed auth log
```

---

## Generate Test Log Data

Before checking Kibana, generate some events so there's data to visualize:

```bash
# Generate SSH failed login events
for i in {1..10}; do
  ssh wronguser@192.168.56.104 -o StrictHostKeyChecking=no 2>/dev/null
done

# Generate Apache access logs
curl http://192.168.56.104/dvwa/
curl "http://192.168.56.104/dvwa/vulnerabilities/sqli/?id=1'+UNION+SELECT+user(),database()--+-"
curl "http://192.168.56.104/dvwa/vulnerabilities/fi/?page=../../../../../../etc/passwd"

# Run Nmap and save XML
sudo nmap -sV 192.168.56.104 -oX /root/nmap-lab-scan.xml

# Check logs were generated
sudo tail -20 /var/log/auth.log
sudo tail -20 /var/log/apache2/access.log
```

---

## Practice Tasks

1. Install Elasticsearch and verify with `curl http://localhost:9200`
2. Install Kibana and access it at `http://localhost:5601`
3. Install Logstash and create the `auth-log.conf` pipeline
4. Create the `apache-log.conf` pipeline
5. Install Filebeat and configure it to ship auth.log and apache2 logs
6. Run `sudo systemctl status elasticsearch logstash kibana filebeat` — confirm all 4 are active
7. Run `curl "http://localhost:9200/_cat/indices?v"` — confirm indices are being created
8. Generate test SSH failed logins: `ssh wronguser@192.168.56.104` × 10
9. Generate test Apache attack logs with the SQLi and LFI curl commands above
10. Open Kibana → Discover → see if log entries appear

---

## Quick Reference

```
SERVICES                        PORTS
Elasticsearch  systemctl        9200  Elasticsearch HTTP
Logstash       systemctl        5044  Beats input
Kibana         systemctl        5601  Kibana web UI
Filebeat       systemctl        5601  → Kibana browser

CONFIG FILES                    LOG FILES
/etc/elasticsearch/             /var/log/elasticsearch/
/etc/logstash/conf.d/           /var/log/logstash/
/etc/kibana/kibana.yml          /var/log/kibana/
/etc/filebeat/filebeat.yml      /var/log/filebeat/

VERIFY COMMANDS
curl http://localhost:9200                    Elasticsearch alive
curl http://localhost:9200/_cat/indices?v     List indices
curl http://localhost:9200/_cat/health        Cluster health
http://localhost:5601                         Kibana browser
```

---

## Resources

- Elastic Installation Guide: https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html
- Logstash Configuration: https://www.elastic.co/guide/en/logstash/current/configuration.html
- Grok Pattern Reference: https://grokdebug.herokuapp.com/
- Filebeat Reference: https://www.elastic.co/guide/en/beats/filebeat/current/index.html
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 25 Complete | Next: Day 26 — Log Ingestion and Index Management*
