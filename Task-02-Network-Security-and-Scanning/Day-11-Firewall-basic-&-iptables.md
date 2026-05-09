# Day 11 — Firewall Basics & iptables
### ApexPlanet Internship | Task 2: Network Security & Scanning
**Timeline:** Days 13–24 | **Author:** Professorshubhx

---

## What is a Firewall

A firewall is a network security device — either hardware, software, or both — that monitors and controls incoming and outgoing network traffic based on predefined rules. It sits between a trusted internal network and an untrusted external network, deciding what gets through and what gets blocked.

Think of it like a security guard at a building entrance. Every person (packet) that wants to enter or leave has to show ID. The guard has a list of rules — who's allowed in, who's blocked, which doors they can use. The firewall does the same thing for network packets.

In Linux, the primary firewall tool is **iptables** — and understanding it is essential for both offensive security (understanding what firewalls block so you can evade them) and defensive security (actually building rules to protect systems).

---

## How iptables Works

iptables works by inspecting packets as they move through the Linux networking stack. It uses three core concepts — **tables**, **chains**, and **rules**.

### Tables

iptables has four tables, each handling a different type of traffic processing:

```
filter    Default table — decides what to ACCEPT, DROP, or REJECT
          This is what you use 99% of the time for firewalling

nat       Network Address Translation — for port forwarding and masquerading
          Used when your Linux machine acts as a router or gateway

mangle    Modifies packet headers — TTL, TOS, marking
          Used for advanced traffic shaping

raw       Bypass connection tracking — for performance in high-traffic systems
```

We focus entirely on the **filter** table today.

### Chains

Each table has chains — points in the packet flow where rules are checked:

```
INPUT     Packets destined FOR this machine
          (someone connecting to your SSH, web server, etc.)

OUTPUT    Packets leaving FROM this machine
          (your machine making outbound connections)

FORWARD   Packets passing THROUGH this machine to another destination
          (only relevant if this machine is a router)
```

Packet flow through the chains:

```
Incoming packet
      │
      ▼
  [PREROUTING]  (nat table)
      │
      ├──► Destined for this machine ──► [INPUT] ──► Local process
      │
      └──► Destined elsewhere ──► [FORWARD] ──► [POSTROUTING] ──► Out

Outgoing packet from local process
      │
      ▼
  [OUTPUT] ──► [POSTROUTING] ──► Out
```

### Rules and Targets

Each rule in a chain specifies match conditions and a target (what to do when a packet matches):

```
ACCEPT    Let the packet through
DROP      Silently discard the packet (sender gets no response)
REJECT    Discard the packet AND send an error back to sender
LOG       Log the packet to syslog (doesn't stop the packet)
RETURN    Stop processing current chain, return to calling chain
```

**DROP vs REJECT:**
- DROP — packet disappears. Sender doesn't know if port is open, closed, or filtered. Makes port scanning slower and harder.
- REJECT — sender gets ICMP "port unreachable" back. More polite but reveals the firewall exists.

In most cases, DROP is preferred for inbound rules. REJECT is useful for outbound rules so your own applications get proper error messages.

---

## Basic iptables Syntax

```bash
iptables [-t table] -COMMAND chain [matches] -j TARGET

-t table    Which table (default is filter)
-A          Append rule to end of chain
-I          Insert rule at beginning of chain (or specific position)
-D          Delete a rule
-L          List all rules
-F          Flush (delete) all rules in chain
-P          Set default policy for chain
-n          Show IPs and ports as numbers (faster, no DNS lookup)
-v          Verbose output (shows packet/byte counters)
--line-numbers  Show rule numbers (needed for -D by number)
```

---

## 1. Viewing Current Rules

```bash
# List all rules in filter table
sudo iptables -L

# List with line numbers (needed to delete specific rules)
sudo iptables -L --line-numbers

# List with numeric output (no DNS resolution, much faster)
sudo iptables -L -n

# List with packet/byte counters (see which rules are being hit)
sudo iptables -L -v -n

# List specific chain only
sudo iptables -L INPUT -n -v
sudo iptables -L OUTPUT -n -v
```

Fresh system with no rules shows:

```
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Default policy ACCEPT means everything is allowed until a rule says otherwise.

---

## 2. Basic Allow and Deny Rules

### Allow Specific Ports

```bash
# Allow SSH (port 22) — always do this before locking down INPUT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP (port 80)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow HTTPS (port 443)
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow DNS (port 53 UDP and TCP)
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT

# Allow a custom port
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

### Block Specific Ports

```bash
# Block Telnet (port 23) — unencrypted, should never be open
sudo iptables -A INPUT -p tcp --dport 23 -j DROP

# Block FTP (port 21) — unencrypted credentials
sudo iptables -A INPUT -p tcp --dport 21 -j DROP

# Block a specific UDP port
sudo iptables -A INPUT -p udp --dport 161 -j DROP

# Block all traffic from a specific IP
sudo iptables -A INPUT -s 192.168.56.100 -j DROP

# Block traffic to a specific destination port from a specific IP
sudo iptables -A INPUT -s 10.0.0.5 -p tcp --dport 80 -j DROP
```

### Allow Traffic from Specific IP

```bash
# Allow all traffic from your management IP
sudo iptables -A INPUT -s 192.168.56.100 -j ACCEPT

# Allow SSH only from a specific IP
sudo iptables -A INPUT -s 192.168.56.100 -p tcp --dport 22 -j ACCEPT

# Allow traffic from entire subnet
sudo iptables -A INPUT -s 192.168.56.0/24 -j ACCEPT
```

### Allow Established Connections

This is a critical rule — without it, blocking INPUT would also block responses to connections you initiated:

```bash
# Allow established and related connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# This means: if YOUR machine started the connection,
# allow the response packets back in
```

Always add this before any DROP rules, otherwise you'll block your own responses.

### Allow Loopback

The loopback interface (lo) is how processes on the same machine communicate. Always allow it:

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
```

---

## 3. Setting Default Policies

Default policies determine what happens to packets that don't match any rule:

```bash
# Drop everything by default (whitelist approach — most secure)
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT   # Usually keep OUTPUT accepting

# WARNING: Run your ACCEPT rules BEFORE setting INPUT to DROP
# Otherwise you'll lock yourself out of SSH
```

**Safe order when locking down a machine:**

```bash
# Step 1: Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Step 2: Allow established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Step 3: Allow specific services you need
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Step 4: THEN set default policy to DROP
sudo iptables -P INPUT DROP

# Now only loopback, established connections, SSH, and HTTP get in
# Everything else is silently dropped
```

---

## 4. Blocking a Port Scan Attempt

One of the Task 2 requirements is demonstrating how to block a port scan. Here's how.

### Detecting Port Scan Traffic

A port scanner like Nmap sends SYN packets to many ports quickly. We can create rules that detect and block this pattern.

```bash
# Block invalid packets (often used in stealth scans)
sudo iptables -A INPUT -m state --state INVALID -j DROP

# Block NULL scans (no flags set — used by Nmap -sN)
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP

# Block XMAS scans (FIN+PSH+URG flags — used by Nmap -sX)
sudo iptables -A INPUT -p tcp --tcp-flags ALL FIN,PSH,URG -j DROP

# Block FIN scans (only FIN flag — used by Nmap -sF)
sudo iptables -A INPUT -p tcp --tcp-flags ALL FIN -j DROP

# Block SYN+FIN combination (never valid in normal traffic)
sudo iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP

# Rate limit new connections to slow down scanning
sudo iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP
# This allows max 1 new TCP connection per second
# Legitimate traffic is fine but a port scan slows dramatically
```

### Block Traffic from Scanning IP

```bash
# If you detect an IP scanning you, block all traffic from it
sudo iptables -A INPUT -s 192.168.56.100 -j DROP

# Verify the block works — from the blocked machine run:
nmap -sS [target-ip]
# All ports should show as filtered
```

### Demonstration — Block and Verify

```bash
# Step 1: On Metasploitable2 (target), add blocking rule
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
sudo iptables -A INPUT -p tcp --tcp-flags ALL FIN,PSH,URG -j DROP

# Step 2: From Kali, run the scans that these rules block
sudo nmap -sN 192.168.56.101    # NULL scan — should be blocked
sudo nmap -sX 192.168.56.101    # XMAS scan — should be blocked
sudo nmap -sF 192.168.56.101    # FIN scan — should be blocked

# All ports will show as filtered rather than open/closed
# Compare with a normal SYN scan which still works:
sudo nmap -sS 192.168.56.101    # SYN scan — still gets through
```

---

## 5. Logging Dropped Packets

Logging is essential for security monitoring. Instead of silently dropping, you can log first:

```bash
# Log and then drop
sudo iptables -A INPUT -p tcp --dport 23 -j LOG --log-prefix "TELNET BLOCKED: " --log-level 4
sudo iptables -A INPUT -p tcp --dport 23 -j DROP

# View the logs
sudo tail -f /var/log/syslog | grep "TELNET BLOCKED"
sudo dmesg | grep "TELNET BLOCKED"

# Log all dropped packets (useful for incident investigation)
sudo iptables -A INPUT -j LOG --log-prefix "INPUT DROP: " --log-level 4
sudo iptables -A INPUT -j DROP
```

Logs look like this in syslog:

```
May 09 10:23:45 kali kernel: TELNET BLOCKED: IN=eth1 OUT= MAC=... SRC=192.168.56.100 DST=192.168.56.101 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=12345 DF PROTO=TCP SPT=54321 DPT=23 WINDOW=64240 RES=0x00 SYN URGP=0
```

Every dropped packet logged with source IP, destination IP, protocol, source port, destination port — perfect for incident investigation.

---

## 6. Managing Rules — Delete, Flush, Save

### Deleting Rules

```bash
# Delete by rule number (view numbers with --line-numbers first)
sudo iptables -L INPUT --line-numbers
sudo iptables -D INPUT 3          # Delete rule number 3 from INPUT

# Delete by matching the exact rule
sudo iptables -D INPUT -p tcp --dport 23 -j DROP

# Flush (delete all rules) from a specific chain
sudo iptables -F INPUT            # Clear all INPUT rules
sudo iptables -F                  # Clear ALL rules in all chains

# Reset everything to default (accept all)
sudo iptables -F
sudo iptables -X                  # Delete user-defined chains
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
```

### Saving Rules

By default, iptables rules are lost on reboot. To persist them:

```bash
# Save current rules
sudo iptables-save > /etc/iptables/rules.v4

# Restore saved rules
sudo iptables-restore < /etc/iptables/rules.v4

# Install iptables-persistent for automatic restore on boot
sudo apt install iptables-persistent -y
# It will ask to save current rules during install — say Yes
```

---

## 7. Complete Practical Firewall Configuration

Here is a complete, realistic firewall setup for a Linux server — build this on your Kali or Metasploitable2 VM:

```bash
#!/bin/bash
# Complete firewall setup script

# Step 1: Flush all existing rules
iptables -F
iptables -X

# Step 2: Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Step 3: Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Step 4: Allow established and related connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Step 5: Allow SSH (change port if you use non-standard)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Step 6: Allow HTTP and HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Step 7: Allow ICMP (ping) — limit to prevent ping floods
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT

# Step 8: Block port scan techniques
iptables -A INPUT -m state --state INVALID -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL FIN,PSH,URG -j DROP
iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP

# Step 9: Rate limit new connections (SYN flood protection)
iptables -A INPUT -p tcp --syn -m limit --limit 25/s --limit-burst 50 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# Step 10: Log everything else before dropping
iptables -A INPUT -j LOG --log-prefix "FIREWALL DROP: " --log-level 4

echo "Firewall rules applied successfully"
iptables -L -v -n
```

Save as `firewall.sh`, make executable, and run:

```bash
chmod +x firewall.sh
sudo ./firewall.sh
```

---

## Practice Tasks

1. View current iptables rules on Kali: `sudo iptables -L -v -n`
2. Add a rule to block port 23 (Telnet): `sudo iptables -A INPUT -p tcp --dport 23 -j DROP`
3. Verify: from Kali try `nc -nv 127.0.0.1 23` — should be blocked
4. Add logging before the drop rule — check `/var/log/syslog` for log entries
5. Block NULL scan packets: `sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP`
6. From Kali, run `sudo nmap -sN 192.168.56.101` — confirm ports show as filtered
7. Run `sudo nmap -sX 192.168.56.101` — XMAS scan should also be filtered after adding that rule
8. Add the rate limiting rule for SYN flood protection
9. Simulate a SYN flood with hping3 and check if rate limiting reduces the impact
10. Flush all rules with `sudo iptables -F` and verify rules are gone with `sudo iptables -L`

---

## Quick Reference

```
VIEWING RULES                   BASIC OPERATIONS
iptables -L -n -v               -A   Append to chain
iptables -L INPUT --line-numbers -I  Insert at top
iptables -L OUTPUT              -D   Delete rule
                                -F   Flush all rules
TARGETS                         -P   Set default policy
ACCEPT  Let through
DROP    Silent discard          COMMON MATCHES
REJECT  Discard + notify        -p tcp/udp
LOG     Log to syslog           --dport 22
                                --sport 1024
ANTI-SCAN RULES                 -s 192.168.x.x
--tcp-flags ALL NONE            -i eth0
--tcp-flags ALL FIN,PSH,URG     -m state --state ESTABLISHED
--tcp-flags SYN,FIN SYN,FIN     -m limit --limit 1/s
-m state --state INVALID
```

---

## Resources

- iptables man page: `man iptables`
- iptables tutorial: https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html
- nftables (modern replacement): https://wiki.nftables.org/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 11 Complete | Task 2 topics covered. Next: Days 12 — Labs, Reports & Deliverables*
