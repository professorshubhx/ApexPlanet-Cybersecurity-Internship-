# Day 07 — Reconnaissance
### ApexPlanet Internship | Task 2: Network Security & Scanning
**Timeline:** Days 13–24 | **Author:** Professorshubhx

---

## What is Reconnaissance

Reconnaissance is the very first phase of any attack or penetration test. Before an attacker touches a single tool or exploits anything, they spend time gathering information about the target. The more they know — IP ranges, employee names, technologies used, open ports, subdomains — the more precise and effective their attack becomes.

As a SOC analyst or security professional, understanding reconnaissance is critical because:
- You need to think like an attacker to defend properly
- Recon activity itself generates logs and alerts you need to recognize
- Many recon techniques are used during authorized security assessments

Reconnaissance is divided into two types — **Passive** and **Active**.

---

## Passive vs Active Reconnaissance

```
Passive Recon                          Active Recon
─────────────────────────────────────────────────────
No direct contact with target          Direct contact with target
Uses publicly available info           Sends packets to target systems
Target cannot detect it                Target may detect it (shows in logs)
Examples: Whois, Google Dorking        Examples: Ping sweep, banner grabbing
Legal in almost all cases              Requires authorization
```

Think of it this way — passive recon is like researching someone on Google before meeting them. Active recon is like calling their phone to see if they pick up.

---

## 1. Passive Reconnaissance

### Whois — Domain Registration Lookup

Whois gives you registration information about a domain — who owns it, when it was registered, which registrar they used, name servers, and sometimes contact details (unless privacy protection is enabled).

```bash
# Command line
whois google.com
whois microsoft.com
whois 8.8.8.8              # Whois on an IP address

# What you get back:
# Registrar name
# Registration and expiry dates
# Name servers (ns1.google.com, ns2.google.com...)
# Registrant contact (often hidden by privacy services)
# Abuse contact email
```

**Online tools:**
- https://who.is/
- https://www.whois.com/
- https://lookup.icann.org/

**What attackers look for in Whois:**
- Expiry dates — expired domains can be hijacked
- Name servers — reveals DNS infrastructure
- Contact emails — targets for phishing
- Organization name — useful for spear phishing

**Practice:**
```bash
whois defronix.com
whois apexplanet.in
# Read through every field — understand what each one means
```

---

### Nslookup — DNS Information Gathering

Nslookup queries DNS servers to map domain names to IPs and discover the DNS infrastructure of a target.

```bash
# Basic lookup
nslookup google.com

# Query specific record types
nslookup -type=MX google.com        # Mail servers
nslookup -type=NS google.com        # Name servers
nslookup -type=TXT google.com       # Text records (SPF, DKIM, verification)
nslookup -type=AAAA google.com      # IPv6 address
nslookup -type=CNAME www.google.com # Canonical name

# Query using a specific DNS server
nslookup google.com 8.8.8.8         # Use Google's DNS
nslookup google.com 1.1.1.1         # Use Cloudflare's DNS

# Reverse lookup (IP to hostname)
nslookup 8.8.8.8
```

**Using dig (more detailed):**

```bash
dig google.com                      # A record
dig google.com MX                   # Mail records
dig google.com NS                   # Name servers
dig google.com TXT                  # Text records
dig google.com ANY                  # All records
dig -x 8.8.8.8                     # Reverse lookup

# Zone transfer attempt (usually blocked but worth trying in lab)
dig axfr @ns1.target.com target.com
# If successful, dumps ALL DNS records — massive info leak
```

**What attackers discover from DNS:**
- Subdomains (mail.company.com, vpn.company.com, dev.company.com)
- Mail server infrastructure (useful for email spoofing assessment)
- Internal IP ranges sometimes exposed in DNS
- Third-party services in use (Cloudflare, AWS, Google Workspace)

---

### Google Dorking — Using Search Engines as a Recon Tool

Google Dorking (also called Google Hacking) uses advanced search operators to find sensitive information that is publicly indexed but not meant to be easily found. It requires zero direct contact with the target — you're just querying Google.

**Core Operators:**

```
site:          Search within a specific domain
               site:microsoft.com

filetype:      Find specific file types
               filetype:pdf site:company.com

intitle:       Page title contains this word
               intitle:"index of" site:company.com

inurl:         URL contains this string
               inurl:admin site:company.com

intext:        Page body contains this text
               intext:"confidential" site:company.com

cache:         Google's cached version of a page
               cache:company.com

link:          Pages linking to this URL
               link:company.com

"quotes"       Exact phrase match
               "internal use only"
```

**Useful Dork Combinations:**

```bash
# Find exposed login pages
site:company.com inurl:login
site:company.com inurl:admin
site:company.com intitle:"login"

# Find exposed files
site:company.com filetype:pdf
site:company.com filetype:xlsx
site:company.com filetype:doc "confidential"

# Find directory listings (misconfigured web servers)
intitle:"index of" site:company.com
intitle:"index of" "parent directory"

# Find exposed configuration files
filetype:env site:company.com
filetype:cfg site:company.com
filetype:conf site:company.com

# Find exposed credentials (be careful — ethical use only)
filetype:txt "username" "password" site:company.com
filetype:log site:company.com

# Find subdomains
site:*.company.com

# Find cameras, routers, and devices
intitle:"webcamXP" inurl:8080
intitle:"router" inurl:admin

# Find SQL errors (indicates potential SQLi)
site:company.com "sql syntax" OR "mysql error"
```

**Google Hacking Database (GHDB):**
The Exploit-DB maintains a database of Google dorks for finding vulnerable systems:
https://www.exploit-db.com/google-hacking-database

**Important:** Google Dorking against targets you don't have permission to test crosses ethical and legal lines even though you're only using Google. In bug bounty, always check scope before dorking a target.

---

### Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)


---

## 2. Active Reconnaissance

Active recon involves sending packets directly to the target. The target may log this activity. Always ensure you have authorization before performing active recon on any system.

In this lab, your target is Metasploitable2 — you own it, so active recon is perfectly fine.

### Ping Sweep — Discovering Live Hosts

A ping sweep sends ICMP Echo Requests to a range of IP addresses to discover which hosts are alive on a network.

```bash
# Ping sweep with Nmap (cleanest method)
nmap -sn 192.168.56.0/24
# -sn means "ping scan only, no port scan"

# Output shows which hosts responded:
# Nmap scan report for 192.168.56.1 (host up)
# Nmap scan report for 192.168.56.101 (host up)

# Ping sweep with fping (faster for large ranges)
sudo apt install fping
fping -a -g 192.168.56.0/24 2>/dev/null
# -a = show alive hosts only
# -g = generate range from CIDR

# Manual ping to single host
ping -c 4 192.168.56.101

# Ping with timestamp (useful for documenting)
ping -c 4 192.168.56.101 | tee ping-results.txt
```

**Why ping sweeps show up in SOC alerts:**
- Sudden ICMP traffic to hundreds of IPs in sequence is a strong recon indicator
- IDS/IPS systems flag this pattern immediately
- Normal users don't ping entire subnets

---

### Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)


---

## Putting It All Together — Recon Workflow

Here's how passive and active recon chain together in a real assessment:

```
Phase 1 — Passive (no target contact)
├── Whois → Find registrar, name servers, contact info
├── Nslookup/dig → Map DNS records, find subdomains
├── Google Dorking → Find exposed files, login pages, errors
└── Shodan → Find internet-facing assets and versions

Phase 2 — Active (direct target contact, authorized only)
├── Ping Sweep → Discover live hosts on network
└── Banner Grabbing → Identify services and versions on live hosts

Output → Target Profile
├── IP ranges
├── Open ports (preliminary)
├── Services and versions
├── Subdomains
├── Technology stack
├── Potential vulnerabilities based on versions found
└── Entry points for next phase (scanning)
```

---

## Practice Tasks

Do all of these before moving to Day 08:

**Passive Recon:**
1. Run `whois google.com` — identify the registrar, name servers, and expiry date
2. Run `dig google.com ANY` — list every DNS record type returned
3. Run `nslookup -type=MX gmail.com` — who handles Google's email?
4. Try this Google dork: `site:github.com "password" filetype:env` — observe what comes up (do not use any credentials found)
5. Create a free Shodan account, search `port:21 vsftpd 2.3.4` — how many results globally?

**Active Recon (Lab only — Metasploitable2):**
6. Run `nmap -sn 192.168.56.0/24` — list every live host discovered
7. Run `nc -nv 192.168.56.101 21` — what exact banner does FTP show?
8. Run `nc -nv 192.168.56.101 22` — what OpenSSH version is running?
9. Run `curl -I http://192.168.56.101` — what web server and version is in the header?
10. Run `nmap -p 21,22,25,80,3306 --script banner 192.168.56.101` — document every banner returned

**Document your findings** in a text file — this becomes your recon report for the deliverable.

---

## Quick Reference

```
PASSIVE RECON                   ACTIVE RECON
whois domain.com                nmap -sn 192.168.x.0/24
dig domain.com ANY              nc -nv host port
nslookup -type=MX domain        curl -I http://host
Google: site:domain.com         nmap --script banner host
Shodan: org:"Company"           fping -a -g 192.168.x.0/24

GOOGLE DORK OPERATORS           SHODAN FILTERS
site:                           hostname:
filetype:                       org:
intitle:                        port:
inurl:                          os:
intext:                         product:
cache:                          version:
```

---

## Resources

- Google Hacking Database: https://www.exploit-db.com/google-hacking-database
- Shodan: https://www.shodan.io/
- DNSDumpster (passive subdomain enum): https://dnsdumpster.com/
- SecurityTrails (DNS history): https://securitytrails.com/
- Whois Lookup: https://who.is/
- Defronix Academy | Mentor: Nitesh Singh Sir

---






---

*Day 07 Complete | Next: Day 08 — Port & Service Scanning (Nmap TCP/UDP, Version Detection, OS Detection)*
