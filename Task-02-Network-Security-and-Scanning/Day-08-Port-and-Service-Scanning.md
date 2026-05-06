# Day 08 — Port & Service Scanning
### ApexPlanet Internship | Task 2: Network Security & Scanning
**Timeline:** Days 13–24 | **Author:** Professorshubhx

---

## What is Port Scanning

After reconnaissance tells you what targets exist, port scanning tells you what's running on them. Every service on a system listens on a port — web servers on 80/443, SSH on 22, FTP on 21. Port scanning systematically checks which ports are open, what services are behind them, and what versions those services are running.

This is the most critical phase before exploitation. A misconfigured or outdated service on an open port is how most real breaches begin.

Today we go deep into Nmap — the tool that does all of this better than anything else.

---

## Quick Recap — Ports

```
Ports range from 0 to 65535

Well-known ports:   0     – 1023   (FTP, SSH, HTTP, HTTPS, DNS...)
Registered ports:   1024  – 49151  (application-specific)
Dynamic/private:    49152 – 65535  (temporary connections)

Common ports to memorize:
21   FTP          53   DNS          443  HTTPS
22   SSH          80   HTTP         3306 MySQL
23   Telnet       110  POP3         3389 RDP
25   SMTP         143  IMAP         8080 HTTP-Alt
```

---

## 1. TCP Scanning

### SYN Scan — The Default

```bash
sudo nmap -sS 192.168.56.101
```

SYN scan is called a "half-open" scan because it never completes the TCP three-way handshake. Here is exactly what happens per port:

```
Nmap sends:   SYN  ──────────────────► Port

If open:      SYN-ACK ◄───────────────  Port  → marked OPEN
              RST     ──────────────────► Port  → connection killed

If closed:    RST ◄─────────────────── Port   → marked CLOSED

If filtered:  [No response or ICMP unreachable] → marked FILTERED
```

Because the handshake is never completed, many older logging systems don't record it — making it stealthier than a full connect scan. This is why it's the default when you run Nmap as root.

### TCP Connect Scan

```bash
nmap -sT 192.168.56.101
```

This completes the full three-way handshake. It doesn't need root privileges because it uses the OS's normal socket API. The downside is it's more detectable — the target application sees a full connection and may log it.

Use this when you don't have root, or when SYN scan is unreliable on the network.

### Comparing SYN vs Connect

```
Feature              SYN Scan (-sS)        Connect Scan (-sT)
Requires root        Yes                   No
Speed                Faster                Slower
Detectability        Lower                 Higher
Completes handshake  No                    Yes
Logged by target     Less likely           More likely
```

---

## 2. UDP Scanning

Most people focus on TCP and forget UDP. That's a mistake — many critical services run on UDP:

```
53   DNS
67   DHCP
69   TFTP
111  RPC
123  NTP
161  SNMP
500  IKE (VPN)
```

```bash
sudo nmap -sU 192.168.56.101
```

UDP scanning is slow because UDP has no handshake — Nmap has to wait for a timeout to determine if a port is filtered vs open. A full UDP scan can take hours on a wide port range.

```bash
# Scan only top UDP ports to save time
sudo nmap -sU --top-ports 20 192.168.56.101

# Combine TCP SYN and UDP in one scan
sudo nmap -sS -sU 192.168.56.101

# Scan specific UDP ports
sudo nmap -sU -p 53,161,500 192.168.56.101
```

UDP port states:

```
open           Got a UDP response back
open|filtered  No response — could be open or firewall dropping packets
closed         Got ICMP port unreachable response
filtered       Got ICMP unreachable but of a different type
```

---

## 3. Service Version Detection

Knowing a port is open isn't enough. You need to know what software is running and what version. That's where `-sV` comes in.

```bash
nmap -sV 192.168.56.101
```

What Nmap does internally:
1. Connects to each open port
2. Sends protocol-specific probes
3. Reads the banner and matches it against its service database
4. Returns: port, protocol, state, service name, version

Sample output against Metasploitable2:

```
PORT     STATE  SERVICE     VERSION
21/tcp   open   ftp         vsftpd 2.3.4
22/tcp   open   ssh         OpenSSH 4.7p1 Debian 8ubuntu1
23/tcp   open   telnet      Linux telnetd
25/tcp   open   smtp        Postfix smtpd
53/tcp   open   domain      ISC BIND 9.4.2
80/tcp   open   http        Apache httpd 2.2.8
139/tcp  open   netbios-ssn Samba smbd 3.X - 4.X
3306/tcp open   mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp open   postgresql  PostgreSQL DB 8.3.0 - 8.3.7
8180/tcp open   http        Apache Tomcat/Coyote JSP 1.1
```

Every one of those versions has known CVEs. This output alone tells you where to start looking for vulnerabilities.

Intensity levels for version detection:

```bash
nmap -sV --version-intensity 0 host    # Lightest — fastest, least accurate
nmap -sV --version-intensity 5 host    # Default
nmap -sV --version-intensity 9 host    # Most aggressive — slowest, most accurate
```

---

## 4. OS Detection

Nmap can fingerprint the operating system of a target by analyzing how it responds to specially crafted TCP/IP packets. Different OS implementations handle things like TTL values, TCP window sizes, and response ordering differently.

```bash
sudo nmap -O 192.168.56.101
```

Sample output:

```
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
```

OS detection requires at least one open and one closed port to work properly. If the target is heavily filtered, results may be inaccurate.

```bash
# More aggressive OS detection attempt
sudo nmap -O --osscan-guess 192.168.56.101
# --osscan-guess tells Nmap to print best guess even if not confident
```

---

## 5. Combining Everything — Practical Scans

### The Standard Reconnaissance Scan

```bash
sudo nmap -sV -sC -O 192.168.56.101
```

This runs version detection, default scripts, and OS detection together. It's what most pentesters run as their second scan after the initial port discovery.

### Full Port Scan

```bash
sudo nmap -p- 192.168.56.101
```

By default Nmap only scans the top 1000 ports. `-p-` scans all 65535. Slower, but catches services running on non-standard ports — which is common in real environments.

### Fast Discovery Then Deep Scan (Two-Phase Approach)

This is the professional approach — get results fast, then go deep:

```bash
# Phase 1: Fast scan — discover all open ports quickly
sudo nmap -p- --min-rate 5000 -oN open-ports.txt 192.168.56.101

# Phase 2: Deep scan — run version/script detection only on open ports
sudo nmap -sV -sC -p 21,22,23,25,53,80,139,3306 192.168.56.101 -oN deep-scan.txt
```

This is much faster than running `-sV -sC -p-` all at once because version detection is only run against ports you know are open.

### Scan a Whole Subnet

```bash
# Discover all live hosts first
sudo nmap -sn 192.168.56.0/24

# Then scan discovered hosts
sudo nmap -sV 192.168.56.1,100,101
```

---

## 6. Nmap Scripting Engine (NSE)

NSE scripts extend Nmap's capabilities into vulnerability detection, brute forcing, information gathering, and more. Scripts are stored at `/usr/share/nmap/scripts/`.

```bash
# Default scripts (safe, commonly used)
nmap -sC 192.168.56.101

# Run a specific script
nmap --script ftp-anon 192.168.56.101
# Checks if FTP allows anonymous login — Metasploitable2 allows this

# Run multiple scripts
nmap --script ftp-anon,ftp-bounce 192.168.56.101

# Run all scripts in a category
nmap --script vuln 192.168.56.101         # All vulnerability scripts
nmap --script auth 192.168.56.101         # All authentication scripts
nmap --script discovery 192.168.56.101    # All discovery scripts

# Wildcard — run all FTP scripts
nmap --script "ftp-*" 192.168.56.101

# Useful individual scripts for Metasploitable2
nmap --script ftp-anon 192.168.56.101             # Anonymous FTP check
nmap --script ssh-brute 192.168.56.101            # SSH brute force
nmap --script http-title 192.168.56.101           # Get web page titles
nmap --script http-enum 192.168.56.101            # Enumerate web directories
nmap --script smb-vuln-ms17-010 192.168.56.101    # EternalBlue check
nmap --script mysql-empty-password 192.168.56.101 # MySQL no password check
nmap --script smtp-commands 192.168.56.101         # SMTP command enum
```

Script categories:

```
auth        Authentication bypass and credential testing
brute       Brute force attacks
default     Run with -sC, safe general purpose scripts
discovery   Service and network information gathering
exploit     Actually exploit vulnerabilities (use with caution)
intrusive   May crash or affect target systems
safe        Won't harm the target
vuln        Check for known vulnerabilities
```

---

## 7. Output Formats and Saving Results

Always save your scan results. You'll need them for your report, and re-scanning wastes time.

```bash
nmap -sV 192.168.56.101 -oN scan.txt       # Normal — human readable
nmap -sV 192.168.56.101 -oX scan.xml       # XML — importable to Metasploit, reports
nmap -sV 192.168.56.101 -oG scan.grep      # Grepable — easy to parse with grep/awk
nmap -sV 192.168.56.101 -oA scan           # All three at once (scan.nmap, .xml, .gnmap)
```

Searching saved grepable output:

```bash
# Find all open ports in a grepable scan
grep "open" scan.grep

# Find hosts with port 80 open
grep "80/open" scan.grep

# Find specific service
grep "ftp" scan.grep
```

---

## 8. Creating the Scan Report

One of your Task 2 deliverables is an Nmap Scan Report. Here is how to structure it properly.

Run this full command against Metasploitable2 and save all output:

```bash
sudo nmap -sV -sC -O -p- --open 192.168.56.101 -oA metasploitable-full-scan
```

Then document your findings in this structure:

```
Target:         192.168.56.101
Hostname:       metasploitable
OS Detected:    Linux 2.6.X
Scan Date:      [date]
Scan Type:      SYN + Version + Default Scripts + OS Detection

OPEN PORTS SUMMARY:
Port   Protocol  Service       Version              Risk
21     TCP        FTP           vsftpd 2.3.4         CRITICAL (backdoor CVE-2011-2523)
22     TCP        SSH           OpenSSH 4.7p1        MEDIUM (outdated version)
23     TCP        Telnet        Linux telnetd         HIGH (plaintext protocol)
25     TCP        SMTP          Postfix smtpd         MEDIUM
53     TCP        DNS           BIND 9.4.2            HIGH (outdated)
80     TCP        HTTP          Apache 2.2.8          HIGH (outdated)
3306   TCP        MySQL         5.0.51a               HIGH (no auth from network)
5432   TCP        PostgreSQL    8.3.0                 MEDIUM
8180   TCP        HTTP          Tomcat 5.5            HIGH (default credentials)

KEY FINDINGS:
1. vsftpd 2.3.4 — Known backdoor on port 6200 (CVE-2011-2523)
2. Telnet running — all traffic including credentials sent in plaintext
3. MySQL accessible from network — potential unauthenticated access
4. Apache Tomcat with likely default credentials (admin/admin, tomcat/tomcat)
5. Anonymous FTP login allowed
```

---

## Timing Templates — Quick Reference

```bash
nmap -T0 host    # Paranoid   — 1 packet per 5 min, IDS evasion
nmap -T1 host    # Sneaky     — slow, avoids detection
nmap -T2 host    # Polite     — reduces bandwidth usage
nmap -T3 host    # Normal     — default
nmap -T4 host    # Aggressive — faster, good for lab use
nmap -T5 host    # Insane     — very fast, may miss results
```

For your lab: use `-T4`. For real authorized engagements: start with `-T2` or `-T3`.

---

## Practice Tasks

Complete all of these on Metasploitable2 before Day 09:

1. Run `sudo nmap -sS 192.168.56.101` — list all open ports found
2. Run `sudo nmap -sU --top-ports 20 192.168.56.101` — any open UDP ports?
3. Run `sudo nmap -sV 192.168.56.101` — document every service and version
4. Run `sudo nmap -O 192.168.56.101` — what OS does Nmap detect?
5. Run `nmap --script ftp-anon 192.168.56.101` — does anonymous FTP work?
6. Run `nmap --script mysql-empty-password 192.168.56.101` — MySQL accessible?
7. Run `nmap --script vuln 192.168.56.101` — what vulnerabilities does NSE find?
8. Run the full scan: `sudo nmap -sV -sC -O -p- --open 192.168.56.101 -oA metasploitable-scan`
9. Open the `.xml` output in a browser — observe the structured format
10. Write your scan report using the template above — this is your Task 2 deliverable

---

## Quick Reference

```
SCAN TYPES                      VERSION & OS
-sS  SYN scan (default)         -sV  Service version detection
-sT  TCP connect scan           -O   OS fingerprinting
-sU  UDP scan                   -A   All of the above + traceroute
-sn  Ping scan only             --version-intensity 0-9

PORT SELECTION                  OUTPUT FORMATS
-p 80,443       Specific ports  -oN  Normal text
-p 1-1000       Range           -oX  XML
-p-             All 65535       -oG  Grepable
--top-ports 100 Top 100         -oA  All formats

NSE SCRIPTS                     TIMING
-sC             Default scripts -T0 Paranoid
--script vuln   Vuln scripts    -T2 Polite
--script auth   Auth scripts    -T4 Aggressive (lab)
--script "ftp-*" All FTP        -T5 Insane
```

---

## Resources

- Nmap Official Book (free): https://nmap.org/book/
- Nmap Script Database: https://nmap.org/nsedoc/
- CVE Details: https://www.cvedetails.com/
- Exploit-DB: https://www.exploit-db.com/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 08 Complete | Next: Day 09 — Vulnerability Scanning (OpenVAS, Nessus, Report Analysis)*
