# Day 04 — Networking Basics
### ApexPlanet Internship | Task 1: Foundation & Environment Setup
**Timeline:** Days 1–12 | **Author:** Professorshubhx

---

## Why Networking Matters in Cybersecurity

Every attack travels over a network. Every alert a SOC analyst investigates involves network traffic. Every tool you'll use — Nmap, Wireshark, Burp Suite, Metasploit — operates at the network level.

If you don't understand how networks work, you won't understand what you're looking at when an alert fires. This is foundational knowledge — not optional.

Today covers the OSI model, TCP/IP, DNS, HTTP/HTTPS, IP addressing, subnetting, and NAT. By the end, you should be able to read a packet capture and understand what's happening at each layer.

---

## 1. The OSI Model

OSI stands for **Open Systems Interconnection**. It's a conceptual framework that breaks down how data travels from one computer to another into 7 distinct layers. Each layer has a specific job and communicates with the layers directly above and below it.

It was created to standardize how different systems communicate — so a Windows machine can talk to a Linux server, or a Cisco router can forward packets from a Juniper switch.

```
Layer 7 — Application    HTTP, HTTPS, FTP, SMTP, DNS, SSH
Layer 6 — Presentation   Encryption, Compression, Encoding (SSL/TLS lives here)
Layer 5 — Session        Manages sessions/connections between applications
Layer 4 — Transport      TCP, UDP — handles end-to-end delivery and ports
Layer 3 — Network        IP — handles logical addressing and routing
Layer 2 — Data Link      MAC addresses, Ethernet, switches, ARP
Layer 1 — Physical       Cables, fiber, Wi-Fi signals, bits on the wire
```

### How to Remember It

Most people use a mnemonic. A common one top-to-bottom:

**A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

(Application, Presentation, Session, Transport, Network, Data Link, Physical)

### What Each Layer Actually Does

**Layer 7 — Application**
This is where the user-facing protocols live. When you type a URL in a browser, HTTP/HTTPS is a Layer 7 protocol. When you send an email, SMTP is Layer 7. This is also where most attacks target — SQL injection, XSS, and phishing all operate at this layer.

**Layer 6 — Presentation**
Responsible for data formatting, encryption, and compression. SSL/TLS encryption happens here before data is handed down to Layer 5. This is why HTTPS traffic is encrypted — the Presentation layer wraps it.

**Layer 5 — Session**
Manages the opening, maintaining, and closing of sessions between two communicating systems. Think of it as the layer that keeps track of "this conversation belongs to this connection." NetBIOS and RPC operate here.

**Layer 4 — Transport**
This is where TCP and UDP live. It handles end-to-end communication, error checking, flow control, and port numbers. When you connect to port 443, that port number is a Layer 4 concept.

**Layer 3 — Network**
IP addressing and routing happen here. Routers operate at Layer 3. When a packet needs to travel from your machine to a server in another country, Layer 3 is responsible for figuring out the path.

**Layer 2 — Data Link**
MAC addresses and switching happen here. When your laptop sends data to your router, the data link layer uses MAC addresses to identify devices on the local network. ARP (Address Resolution Protocol) maps IP addresses to MAC addresses and lives here.

**Layer 1 — Physical**
The raw bits traveling over a physical medium — copper cable, fiber optic, radio waves for Wi-Fi. If a cable is unplugged, that's a Layer 1 problem.

### Why SOC Analysts Care About OSI

When you investigate an alert, you need to know which layer the attack is targeting:

```
Layer 7 attack  — SQL Injection, XSS, phishing, API abuse
Layer 4 attack  — Port scanning, SYN flood (DDoS)
Layer 3 attack  — IP spoofing, routing attacks, ICMP flood
Layer 2 attack  — ARP spoofing, MAC flooding
Layer 1 attack  — Physical cable tampering, rogue devices
```

When a Wireshark capture lands on your desk, you're looking at all layers simultaneously. Understanding OSI lets you know where to focus.

---

## 2. TCP/IP Protocol Suite

While OSI is a theoretical model, TCP/IP is what the internet actually runs on. It's a 4-layer model that maps roughly to OSI:

```
TCP/IP Layer          OSI Equivalent          Protocols
Application           Layers 5, 6, 7          HTTP, HTTPS, DNS, FTP, SSH, SMTP
Transport             Layer 4                 TCP, UDP
Internet              Layer 3                 IP, ICMP, ARP
Network Access        Layers 1, 2             Ethernet, Wi-Fi, MAC
```

### TCP — Transmission Control Protocol

TCP is connection-oriented. Before any data is sent, a connection is established using the **three-way handshake**:

```
Client                          Server
  |                               |
  |-------- SYN ----------------->|   "I want to connect"
  |                               |
  |<------- SYN-ACK --------------|   "OK, I acknowledge, are you ready?"
  |                               |
  |-------- ACK ----------------->|   "Yes, I'm ready"
  |                               |
  |====== Data Transfer ========= |
  |                               |
  |-------- FIN ----------------->|   "I'm done"
  |<------- FIN-ACK --------------|
```

TCP guarantees delivery. If a packet is lost, TCP will retransmit it. This makes it reliable but slightly slower than UDP.

**TCP is used for:** HTTP/HTTPS, SSH, FTP, SMTP, anything where you can't afford to lose data.

**The SYN Flood Attack** exploits the three-way handshake. An attacker sends thousands of SYN packets but never completes the handshake. The server keeps allocating resources waiting for the ACK that never comes, eventually running out of capacity. This is a classic DDoS technique.

### UDP — User Datagram Protocol

UDP is connectionless. It just fires packets at the destination with no handshake, no guarantee of delivery, no retransmission. It's faster than TCP but unreliable.

**UDP is used for:** DNS queries, video streaming, online gaming, VoIP — situations where speed matters more than perfect delivery. A dropped frame in a video call is better than freezing while waiting for a retransmit.

### TCP vs UDP at a Glance

```
Feature             TCP                 UDP
Connection          Required (3-way)    None
Reliability         Guaranteed          Best-effort
Speed               Slower              Faster
Order               Maintains order     No order guarantee
Use cases           HTTP, SSH, FTP      DNS, streaming, gaming
Header size         20 bytes            8 bytes
```

### ICMP — Internet Control Message Protocol

ICMP isn't for data transfer — it's for network diagnostics and error reporting. The `ping` command uses ICMP Echo Request and Echo Reply messages.

```bash
ping 8.8.8.8        # Sends ICMP Echo Request, expects Echo Reply
traceroute host     # Uses ICMP (or UDP) to map the route to a destination
```

Attackers use ICMP for reconnaissance (ping sweeps) and sometimes for covert data exfiltration (ICMP tunneling).

---

## 3. IP Addressing

Every device on a network has an IP address. Think of it like a postal address — it tells the network where to deliver packets.

### IPv4

IPv4 addresses are 32 bits long, written as four octets separated by dots:

```
192.168.1.100

192     .168    .1      .100
11000000.10101000.00000001.01100100  (in binary)
```

Each octet is 8 bits, so each ranges from 0 to 255.

### IPv4 Address Classes

```
Class A:  1.0.0.0    to  126.255.255.255   /8   Large organizations
Class B:  128.0.0.0  to  191.255.255.255   /16  Medium organizations
Class C:  192.0.0.0  to  223.255.255.255   /24  Small networks
Class D:  224.0.0.0  to  239.255.255.255   Multicast
Class E:  240.0.0.0  to  255.255.255.255   Reserved/Experimental
```

### Private vs Public IP Addresses

Private IPs are used inside local networks (your home, office, lab). They are not routable on the internet. NAT (covered below) is what allows private IPs to access the internet.

```
Private ranges:
10.0.0.0     to  10.255.255.255    (10.0.0.0/8)
172.16.0.0   to  172.31.255.255   (172.16.0.0/12)
192.168.0.0  to  192.168.255.255  (192.168.0.0/16)

Special:
127.0.0.1   Loopback (localhost) — your own machine
0.0.0.0     Default route / any address
255.255.255.255  Broadcast
```

Your Kali VM and Metasploitable2 both use `192.168.56.x` addresses — these are private addresses on the Host-Only network.

### IPv6

IPv6 was created because the world ran out of IPv4 addresses (only ~4.3 billion possible). IPv6 is 128 bits long, written in hexadecimal:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

Can be shortened by removing leading zeros and replacing consecutive zero groups with `::`:

```
2001:db8:85a3::8a2e:370:7334
```

IPv6 loopback is `::1` (equivalent to `127.0.0.1` in IPv4).

---

## 4. Subnetting

Subnetting is dividing a large network into smaller sub-networks. It improves security (containing breaches), performance (reducing broadcast traffic), and organization.

### Subnet Mask

A subnet mask tells you which part of an IP address is the network portion and which part is the host portion.

```
IP Address:    192.168.1.100
Subnet Mask:   255.255.255.0
In binary:
IP:   11000000.10101000.00000001.01100100
Mask: 11111111.11111111.11111111.00000000
                                ^^^^^^^^^
                                This part = host addresses
```

The `1`s in the mask = network portion. The `0`s = host portion.

### CIDR Notation

Instead of writing `255.255.255.0`, we use CIDR (Classless Inter-Domain Routing) notation — just count the `1` bits in the mask:

```
255.255.255.0   = /24  (24 ones in the mask)
255.255.0.0     = /16
255.0.0.0       = /8
255.255.255.128 = /25
255.255.255.192 = /26
```

### Calculating Hosts Per Subnet

Formula: **2^(host bits) - 2**

The `-2` is because one address is the network address and one is the broadcast address.

```
/24 subnet: 32 - 24 = 8 host bits → 2^8 - 2 = 254 usable hosts
/25 subnet: 32 - 25 = 7 host bits → 2^7 - 2 = 126 usable hosts
/26 subnet: 32 - 26 = 6 host bits → 2^6 - 2 = 62 usable hosts
/30 subnet: 32 - 30 = 2 host bits → 2^2 - 2 = 2 usable hosts (point-to-point links)
```

### Subnetting Example

Network: `192.168.1.0/24`

```
Network address:   192.168.1.0     (not assignable)
First host:        192.168.1.1
Last host:         192.168.1.254
Broadcast:         192.168.1.255   (not assignable)
Usable hosts:      254
```

### Why Subnetting Matters for Security

Network segmentation (putting different systems in different subnets) is a core security control. If malware infects one subnet, proper segmentation stops it from spreading to others. This is why enterprise networks have separate subnets for servers, workstations, IoT devices, and guest Wi-Fi.

---

## 5. NAT — Network Address Translation

NAT allows multiple devices with private IP addresses to share a single public IP address when accessing the internet. Your home router does this constantly.

```
Your devices (private IPs):
Laptop:  192.168.1.10  ─┐
Phone:   192.168.1.11  ─┤──► Router (NAT) ──► 203.0.113.5 (public IP) ──► Internet
Tablet:  192.168.1.12  ─┘
```

When your laptop sends a request to google.com, the router:
1. Replaces the source IP `192.168.1.10` with `203.0.113.5`
2. Records the translation in a NAT table
3. When Google's response comes back, the router looks up the table and forwards it to `192.168.1.10`

From Google's perspective, every device in your house is the same IP address.

### Types of NAT

**Static NAT** — one private IP maps permanently to one public IP. Used for servers that need to be reachable from the internet.

**Dynamic NAT** — a pool of public IPs assigned dynamically to private IPs as needed.

**PAT (Port Address Translation) / NAT Overload** — what your home router uses. Many private IPs share one public IP, differentiated by port numbers. Also called NAPT.

### NAT and Security Investigations

NAT complicates incident response. If your logs show an attack from `203.0.113.5`, that IP might represent dozens of users behind a router. Conversely, if a private IP appears in logs, you need to know which NAT device translated it to track the actual source.

---

## 6. DNS — Domain Name System

DNS translates human-readable domain names into IP addresses. It's the phone book of the internet.

When you type `google.com` in your browser:

```
1. Your machine checks its local cache — has it looked this up recently?
2. If not, asks your configured DNS resolver (usually your router or ISP)
3. Resolver asks a Root DNS server — "who handles .com?"
4. Root server points to the .com TLD (Top Level Domain) server
5. TLD server points to Google's authoritative DNS server
6. Google's DNS server returns: google.com = 142.250.x.x
7. Your browser connects to that IP
```

This whole process typically takes under 100ms.

### DNS Record Types

```
A        Maps hostname to IPv4 address
         google.com → 142.250.x.x

AAAA     Maps hostname to IPv6 address
         google.com → 2607:f8b0:4004::x

CNAME    Canonical Name — alias for another hostname
         www.google.com → google.com

MX       Mail Exchange — where to send email for this domain
         google.com MX → smtp.google.com

TXT      Text records — used for verification, SPF, DKIM
         google.com TXT → "v=spf1 include:_spf.google.com ~all"

NS       Name Server — which servers are authoritative for this domain
         google.com NS → ns1.google.com

PTR      Reverse lookup — IP address to hostname
         142.250.x.x → google.com

SOA      Start of Authority — info about the zone itself
```

### DNS Commands in Practice

```bash
# Basic lookups
nslookup google.com
dig google.com
dig google.com A              # Specifically query A records
dig google.com MX             # Mail records
dig google.com TXT            # Text records
dig -x 8.8.8.8               # Reverse lookup (PTR record)

# Query a specific DNS server
dig @8.8.8.8 google.com       # Use Google's DNS
dig @1.1.1.1 google.com       # Use Cloudflare's DNS

# Zone transfer attempt (often blocked, but worth knowing)
dig axfr @ns1.target.com target.com
```

### DNS from a Security Perspective

DNS is frequently abused by attackers:

- **DNS Spoofing / Cache Poisoning** — injecting fake DNS records so users get sent to attacker-controlled IPs
- **DNS Tunneling** — encoding data inside DNS queries to exfiltrate data or communicate with C2 servers while bypassing firewalls
- **Subdomain Takeover** — claiming an abandoned subdomain that still has a DNS record pointing to it
- **DNS Reconnaissance** — using DNS to map out a target organization's infrastructure before an attack

As a SOC analyst, unusual DNS queries (high volume, strange domains, long query names) are a key indicator of compromise.

---

## 7. HTTP and HTTPS

HTTP (HyperText Transfer Protocol) is the foundation of data exchange on the web. HTTPS is HTTP with TLS encryption on top.

### HTTP Request/Response Cycle

```
Client (Browser)                          Server
      |                                     |
      |--- GET /index.html HTTP/1.1 ------->|
      |    Host: example.com                |
      |    User-Agent: Mozilla/5.0          |
      |    Accept: text/html                |
      |                                     |
      |<-- HTTP/1.1 200 OK ----------------|
      |    Content-Type: text/html          |
      |    Content-Length: 1234             |
      |                                     |
      |    <html>...</html>                 |
```

### HTTP Methods

```
GET      Retrieve a resource (should not change server state)
POST     Submit data to the server (login forms, file uploads)
PUT      Update/replace a resource
DELETE   Delete a resource
PATCH    Partially update a resource
HEAD     Like GET but returns only headers, no body
OPTIONS  Ask server what methods are allowed
```

From a security perspective, understanding methods matters because:
- PUT and DELETE should almost never be exposed publicly
- POST parameters are a common injection point
- Misconfigured OPTIONS can reveal server capabilities to attackers

### HTTP Status Codes

```
2xx — Success
  200 OK              Request succeeded
  201 Created         Resource created (after POST/PUT)

3xx — Redirection
  301 Moved Permanently   Resource has moved
  302 Found               Temporary redirect

4xx — Client Errors
  400 Bad Request         Malformed request
  401 Unauthorized        Authentication required
  403 Forbidden           Authenticated but no permission
  404 Not Found           Resource doesn't exist
  429 Too Many Requests   Rate limited

5xx — Server Errors
  500 Internal Server Error   Something broke on the server
  502 Bad Gateway             Upstream server issue
  503 Service Unavailable     Server overloaded or down
```

In Burp Suite and web app testing, you'll read status codes constantly.

### HTTP vs HTTPS

```
HTTP    Port 80    Plaintext — anyone on the network can read traffic
HTTPS   Port 443   Encrypted with TLS — traffic is unreadable to interceptors
```

HTTPS uses TLS (Transport Layer Security) to:
- Encrypt the data in transit
- Verify the server's identity using digital certificates
- Prevent tampering with the content

When you see a padlock in the browser, TLS is working. When you capture HTTPS traffic in Wireshark, you see encrypted gibberish — unless you have the server's private key or the session keys.

### Common HTTP Headers (Security Relevant)

```
Request Headers:
  Authorization: Bearer <token>      Authentication token
  Cookie: session=abc123             Session identifier
  User-Agent: Mozilla/5.0            Client software info
  Referer: https://example.com       Where the request came from
  X-Forwarded-For: 1.2.3.4          Real IP behind a proxy

Response Headers:
  Set-Cookie: session=abc123         Server setting a cookie
  Content-Security-Policy:           Prevents XSS
  X-Frame-Options: DENY              Prevents clickjacking
  Strict-Transport-Security:         Forces HTTPS
  Access-Control-Allow-Origin:       CORS policy
```

Missing security headers is a common finding in web application security assessments.

---

## Putting It All Together — What Happens When You Visit a Website

Let's trace a complete request to `https://google.com` through all the concepts covered today:

```
1. DNS Resolution (Application Layer)
   Your machine queries DNS for google.com
   Gets back: 142.250.x.x

2. TCP Three-Way Handshake (Transport Layer)
   SYN → SYN-ACK → ACK with 142.250.x.x:443

3. TLS Handshake (Presentation Layer)
   Browser and server negotiate encryption
   Server presents certificate
   Session keys are established

4. HTTP GET Request (Application Layer)
   GET / HTTP/1.1
   Host: google.com

5. IP Routing (Network Layer)
   Packets travel through routers
   Each router forwards based on destination IP

6. NAT Translation (Network Layer)
   Your router replaces your private IP with public IP

7. Response Returns
   Google sends back HTML
   Travels same path in reverse
   NAT table maps it back to your device
   TLS decrypts it
   Browser renders the page
```

---

## Practice Tasks

Do these before moving to Day 05:

1. Run `ip a` on Kali — identify which interface is on the Host-Only network and what its IP is
2. Run `ip route` — identify your default gateway
3. Use `dig google.com` — identify the A record returned
4. Use `dig google.com MX` — who handles Google's email?
5. Use `nmap -sV 192.168.56.101` on Metasploitable2 — list every open port and service
6. Open Wireshark on Kali, start capturing on the Host-Only interface, ping Metasploitable2, then stop and look at the ICMP packets
7. Calculate: how many usable hosts does a `/26` subnet give you?
8. What private IP range does your lab use? What class is it?
9. Run `curl -I https://google.com` — look at the response headers returned
10. In Wireshark, filter by `dns` — what DNS queries does Kali make just sitting idle?

---

## Quick Reference

```
OSI LAYERS (top to bottom)        TCP/IP LAYERS
7 - Application (HTTP, DNS)       Application
6 - Presentation (TLS)            Application
5 - Session                       Application
4 - Transport (TCP, UDP)          Transport
3 - Network (IP)                  Internet
2 - Data Link (MAC, ARP)          Network Access
1 - Physical (cables, signals)    Network Access

SUBNETTING                        DNS RECORDS
/24 = 254 hosts                   A     → IPv4
/25 = 126 hosts                   AAAA  → IPv6
/26 = 62 hosts                    MX    → Mail
/27 = 30 hosts                    CNAME → Alias
/28 = 14 hosts                    TXT   → Text/SPF
/30 = 2 hosts                     PTR   → Reverse

HTTP STATUS CODES
200 OK | 301 Redirect | 401 Unauth | 403 Forbidden | 404 Not Found | 500 Server Error
```

---

## Resources

- Subnet Calculator: https://www.subnet-calculator.com/
- Wireshark Sample Captures: https://wiki.wireshark.org/SampleCaptures
- DNS Lookup Tool: https://mxtoolbox.com/
- HTTP Headers Reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- YouTube: [@Professorshubhx](https://tryhackme.com/p/professorshubhx)
- LinkedIn: [Professorshubhx](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)

---

*Day 04 Complete | Next: Day 05 — Cryptography Basics (Encryption, Hashing, SSL/TLS, OpenSSL)*
