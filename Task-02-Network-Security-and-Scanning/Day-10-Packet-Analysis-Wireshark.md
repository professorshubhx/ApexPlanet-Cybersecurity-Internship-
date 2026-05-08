# Day 10 — Packet Analysis with Wireshark
### ApexPlanet Internship | Task 2: Network Security & Scanning
**Timeline:** Days 13–24 | **Author:** Professorshubhx

---

## Why Packet Analysis Matters

Every attack leaves a trail in network traffic. A SOC analyst's job is to look at that traffic and figure out what happened — was this normal browsing or data exfiltration? Was this a login attempt or a brute force? Was this DNS traffic or DNS tunneling?

Wireshark lets you see exactly what's moving across a network at the packet level. Today we go beyond the basics and do four things the task specifically requires:

- Capture HTTP, FTP, and DNS traffic
- Filter and extract credentials from unencrypted protocols
- Simulate and analyze a SYN flood attack using hping3
- Understand what each attack looks like in a packet capture

All exercises use your Kali VM and Metasploitable2.

---

## Setup Before Starting

```bash
# Make sure Wireshark is installed
sudo apt install wireshark -y

# Allow non-root capture (so you don't always need sudo)
sudo usermod -aG wireshark $USER
# Log out and back in for this to take effect

# Start Wireshark
wireshark &

# Or with sudo if needed
sudo wireshark &
```

Select your **Host-Only network interface** (eth1 or the one with 192.168.56.x IP) before starting any capture.

---

## 1. Capturing HTTP Traffic

HTTP is plaintext — everything sent over port 80 is visible in a packet capture. This includes usernames, passwords, cookies, form data, and any other information submitted through a web form.

### Exercise — Capture DVWA Login

```
Step 1: Start Wireshark capture on Host-Only interface
Step 2: Open Kali browser → go to http://192.168.56.101/dvwa/login.php
Step 3: Login with username: admin  password: password
Step 4: Stop capture in Wireshark
Step 5: Apply filter: http
```

Now look for the POST request. It will look like this in the packet list:

```
No.   Time      Source           Destination      Protocol  Info
47    2.341     192.168.56.100   192.168.56.101   HTTP      POST /dvwa/login.php HTTP/1.1
```

Click on that packet. In the Packet Details pane, expand:
```
Hypertext Transfer Protocol
  └── HTML Form URL Encoded: application/x-www-form-urlencoded
        ├── username: admin
        ├── password: password
        └── Login: Login
```

The credentials are right there — completely readable. This is exactly why HTTP login pages are dangerous and why everything should use HTTPS.

### Following the HTTP Stream

Right-click any HTTP packet → Follow → HTTP Stream

You'll see the complete conversation:

```
GET /dvwa/login.php HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0...

HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: PHPSESSID=abc123; path=/

POST /dvwa/login.php HTTP/1.1
...
username=admin&password=password&Login=Login

HTTP/1.1 302 Found
Location: index.php
Set-Cookie: security=low
```

This stream shows the full request-response cycle including the session cookie being set after login.

### Useful HTTP Filters

```bash
http                                    # All HTTP traffic
http.request                            # Only requests
http.response                           # Only responses
http.request.method == "POST"           # Only POST requests
http.request.method == "GET"            # Only GET requests
http.response.code == 200               # Successful responses
http.response.code == 401               # Unauthorized responses
http.response.code == 302               # Redirects
http.request.uri contains "login"       # Requests to login pages
http.request.uri contains "admin"       # Requests to admin pages
http.cookie contains "session"          # Requests with session cookies
```

---

## 2. Capturing FTP Traffic

FTP is one of the most dangerous protocols still in use — it sends everything in plaintext including credentials. Telnet does the same. Any attacker on the same network segment can capture your FTP login.

### Exercise — Capture FTP Credentials

```bash
# Step 1: Start Wireshark capture
# Step 2: From Kali terminal, connect to Metasploitable2 FTP
ftp 192.168.56.101

# When prompted:
Name: msfadmin
Password: msfadmin

# Run a few commands
ls
pwd
bye

# Step 3: Stop Wireshark capture
# Step 4: Apply filter: ftp
```

In Wireshark you will see the FTP control channel conversation:

```
No.  Protocol  Info
1    FTP        Response: 220 (vsFTPd 2.3.4)
2    FTP        Request: USER msfadmin
3    FTP        Response: 331 Please specify the password.
4    FTP        Request: PASS msfadmin
5    FTP        Response: 230 Login successful.
6    FTP        Request: PWD
7    FTP        Response: 257 "/" is the current directory
```

The username and password are in plain sight. No decryption needed — it's already readable text.

### Following the FTP Stream

Right-click any FTP packet → Follow → TCP Stream

```
220 (vsFTPd 2.3.4)
USER msfadmin
331 Please specify the password.
PASS msfadmin
230 Login successful.
SYST
215 UNIX Type: L8
PWD
257 "/" is the current directory.
QUIT
221 Goodbye.
```

Complete session reconstructed in seconds.

### FTP Data Channel

FTP uses two channels — a control channel (port 21) and a data channel (port 20 or a high port in passive mode). When you do `ls` or download a file, the actual data travels on the data channel.

```bash
# Filter both channels
ftp || ftp-data

# To see file contents being transferred
ftp-data
# Then follow the TCP stream to see the actual file data
```

### Useful FTP Filters

```bash
ftp                                 # All FTP control traffic
ftp-data                            # FTP data transfer traffic
ftp || ftp-data                     # Both combined
ftp.request.command == "USER"       # Username submissions
ftp.request.command == "PASS"       # Password submissions
ftp.request.command == "RETR"       # File download commands
ftp.request.command == "STOR"       # File upload commands
ftp.response.code == 230            # Successful logins
ftp.response.code == 530            # Failed logins
```

---

## 3. Capturing DNS Traffic

DNS queries happen constantly in the background — every domain name lookup generates DNS traffic. Analyzing DNS traffic helps you:

- See what domains a machine is communicating with
- Detect DNS tunneling (data hidden in DNS queries)
- Identify C2 (command and control) communication
- Troubleshoot connectivity issues

### Exercise — Capture DNS Queries

```bash
# Step 1: Start Wireshark capture
# Step 2: From Kali terminal, run DNS queries
dig google.com @8.8.8.8
dig facebook.com @8.8.8.8
nslookup microsoft.com
nslookup -type=MX gmail.com

# Step 3: Stop capture
# Step 4: Apply filter: dns
```

Each DNS query and response pair looks like this:

```
No.  Protocol  Info
1    DNS       Standard query 0x1234 A google.com
2    DNS       Standard query response 0x1234 A google.com A 142.250.x.x
```

Click the response packet. Expand in Packet Details:

```
Domain Name System (response)
  └── Answers
        └── google.com: type A, class IN, addr 142.250.x.x
              Name: google.com
              Type: A (Host Address)
              TTL: 300
              Address: 142.250.x.x
```

### Spotting Suspicious DNS — What SOC Analysts Look For

```bash
# Long domain names in DNS queries (possible DGA or DNS tunneling)
dns.qry.name matches "[a-z0-9]{20,}\..*"

# High volume DNS queries to same domain (possible tunneling)
dns.qry.name contains "suspicious-domain.com"

# DNS queries with unusually large payload (data in DNS)
dns && frame.len > 512

# Failed DNS lookups (NXDOMAIN — domain doesn't exist)
dns.flags.rcode == 3

# DNS over non-standard port (evasion technique)
dns.port != 53 && dns
```

**DNS Tunneling indicators:**
- Very long subdomains (data encoded as subdomain labels)
- High frequency queries to same base domain
- TXT record queries more than A record queries
- Large DNS response sizes

### Useful DNS Filters

```bash
dns                                 # All DNS traffic
dns.qry.name contains "google"      # Queries for google
dns.flags.response == 0             # Only queries (not responses)
dns.flags.response == 1             # Only responses
dns.qry.type == 1                   # A record queries only
dns.qry.type == 15                  # MX record queries
dns.flags.rcode == 0                # Successful responses
dns.flags.rcode == 3                # NXDOMAIN (domain not found)
```

---

## 4. Simulating and Analyzing a SYN Flood Attack

A SYN flood is a type of DDoS attack that exploits the TCP three-way handshake. The attacker sends thousands of SYN packets with spoofed source IPs. The target responds with SYN-ACK to each one and waits for an ACK that never comes, exhausting its connection table.

We simulate this using **hping3** — a powerful packet crafting tool on Kali.

### Installing hping3

```bash
sudo apt install hping3 -y
```

### Exercise — SYN Flood Simulation

```bash
# Step 1: Start Wireshark capture on Host-Only interface
# Apply capture filter before starting: host 192.168.56.101

# Step 2: In a separate terminal, launch the SYN flood
sudo hping3 -S --flood -V -p 80 192.168.56.101

# Flag breakdown:
# -S        Set SYN flag
# --flood   Send packets as fast as possible
# -V        Verbose output
# -p 80     Target port 80 (HTTP)

# Run for 10-15 seconds then Ctrl+C to stop
# Step 3: Stop Wireshark capture
```

### What You'll See in Wireshark

The packet list will be flooded with SYN packets:

```
No.    Source           Destination      Protocol  Info
1      192.168.56.100   192.168.56.101   TCP       [SYN] Seq=0 Win=512
2      192.168.56.100   192.168.56.101   TCP       [SYN] Seq=0 Win=512
3      192.168.56.100   192.168.56.101   TCP       [SYN] Seq=0 Win=512
4      192.168.56.101   192.168.56.100   TCP       [SYN, ACK] Seq=0 Ack=1
5      192.168.56.100   192.168.56.101   TCP       [SYN] Seq=0 Win=512
...thousands more...
```

Notice that Metasploitable2 responds with SYN-ACK to each one (packet 4), but Kali never sends the final ACK. In a real attack with thousands of spoofed source IPs, the target would keep sending SYN-ACK responses and filling its connection table — eventually unable to accept legitimate connections.

### Filters to Analyze the SYN Flood

```bash
# See only SYN packets (the attack traffic)
tcp.flags.syn == 1 && tcp.flags.ack == 0

# See only SYN-ACK responses (target responding to flood)
tcp.flags.syn == 1 && tcp.flags.ack == 1

# Count the volume — Statistics → IO Graph with these filters
# tcp.flags.syn == 1 && tcp.flags.ack == 0   (attack line)
# tcp.flags.syn == 1 && tcp.flags.ack == 1   (response line)
```

### Using Wireshark Statistics to Visualize the Attack

After capturing the SYN flood:

```
Statistics → IO Graph
  Add two lines:
  Line 1: tcp.flags.syn == 1 && tcp.flags.ack == 0   (color: red)
  Line 2: tcp.flags.syn == 1 && tcp.flags.ack == 1   (color: blue)

You'll see:
  Red line spikes massively during the flood
  Blue line follows (target trying to respond)
  Both drop to zero when attack stops
```

This visualization is exactly what a SOC analyst would show in an incident report.

### Using Statistics → Conversations

```
Statistics → Conversations → TCP tab
  You'll see thousands of half-open connections
  All from same source IP (192.168.56.100 in our lab)
  In real attacks — thousands of different spoofed source IPs
```

### SYN Flood with Random Source IPs (More Realistic)

```bash
sudo hping3 -S --flood -V -p 80 --rand-source 192.168.56.101
# --rand-source randomizes the source IP
# Makes it look more like a real botnet-based DDoS
```

With `--rand-source`, the Wireshark capture will show packets coming from thousands of different source IPs — which is how a real DDoS looks.

### How to Detect a SYN Flood in SOC

In a real environment, these are the indicators:

```
Network level:
  Massive spike in inbound SYN packets
  Large number of half-open connections (SYN_RECV state)
  Target system becomes slow or unresponsive
  Bandwidth consumption spikes

In Wireshark/packet capture:
  SYN packets massively outnumber SYN-ACK + ACK packets
  Many different source IPs (if spoofed) or same IP (if not)
  Short window sizes (hping3 default is 512 or 64)
  No valid TCP conversations completing

Linux command to see half-open connections:
  ss -ant | grep SYN_RECV | wc -l
  netstat -ant | grep SYN_RECV
```

---

## 5. Combining All Three — A Realistic Investigation Scenario

Here is how all of today's skills come together in a real SOC scenario:

```
Alert fires: "Unusual traffic from internal host 192.168.56.100"

Step 1: Start packet capture, filter by source IP
        ip.src == 192.168.56.100

Step 2: Check protocol breakdown
        Statistics → Protocol Hierarchy
        See large % of TCP SYN packets → possible scanning or SYN flood

Step 3: Filter for SYN packets specifically
        tcp.flags.syn == 1 && tcp.flags.ack == 0
        Thousands of SYNs to same destination → SYN flood confirmed

Step 4: Check if there's also HTTP traffic
        http && ip.src == 192.168.56.100
        Found POST requests to /login → possible credential stuffing

Step 5: Follow HTTP streams
        Right-click → Follow → HTTP Stream
        See username/password pairs being submitted → brute force confirmed

Step 6: Check DNS traffic
        dns && ip.src == 192.168.56.100
        Long subdomain queries to unknown domain → possible DNS exfiltration

Conclusion: This host is conducting a SYN flood, brute-forcing a web login,
and possibly exfiltrating data via DNS tunneling.
Recommended action: Isolate host immediately, preserve logs, begin forensics.
```

---

## Practice Tasks

1. Start Wireshark and capture a DVWA HTTP login — find the POST request and read the credentials from the packet
2. Follow the HTTP stream from the login — identify the session cookie set after login
3. Connect to Metasploitable2 via FTP from Kali — capture the full session and read credentials
4. Filter for `ftp.request.command == "PASS"` — confirm password is visible in plaintext
5. Run `dig google.com @8.8.8.8` while capturing — find the A record in the DNS response packet
6. Filter `dns.flags.rcode == 3` — query a non-existent domain and observe the NXDOMAIN response
7. Run the hping3 SYN flood for 15 seconds while capturing — filter `tcp.flags.syn == 1 && tcp.flags.ack == 0` and count the packets
8. Open Statistics → IO Graph during the SYN flood capture — visualize attack vs response traffic
9. Run `sudo hping3 -S --flood -V -p 80 --rand-source 192.168.56.101` — observe how different source IPs appear
10. Use Statistics → Protocol Hierarchy on any of your captures — document the percentage breakdown of protocols

---

## Quick Reference

```
HTTP FILTERS                    FTP FILTERS
http                            ftp
http.request.method == "POST"   ftp-data
http.response.code == 200       ftp.request.command == "PASS"
http.request.uri contains "x"   ftp.response.code == 230

DNS FILTERS                     SYN FLOOD FILTERS
dns                             tcp.flags.syn == 1 && tcp.flags.ack == 0
dns.qry.name contains "x"       tcp.flags.syn == 1 && tcp.flags.ack == 1
dns.flags.rcode == 3            tcp.flags.rst == 1
dns.flags.response == 0

HPING3 FLAGS                    WIRESHARK STATISTICS
-S    SYN flag                  Statistics → IO Graph
-A    ACK flag                  Statistics → Protocol Hierarchy
-p    Target port               Statistics → Conversations
-i    Interval between packets  Analyze → Expert Information
--flood  Send as fast as possible
--rand-source  Random source IP
```

---

## Resources

- Wireshark Sample Captures: https://wiki.wireshark.org/SampleCaptures
- Wireshark Display Filter Reference: https://www.wireshark.org/docs/dfref/
- hping3 Manual: http://www.hping.org/manpage.html
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 10 Complete | Next: Day 11 — Firewall Basics (iptables rules, blocking port scans)*
