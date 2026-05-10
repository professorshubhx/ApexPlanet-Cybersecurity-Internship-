# Nmap Scan Report
### ApexPlanet Internship | Task 2 Deliverable
## Report by: Professorshubhx 

**Target:** Metasploitable2 (192.168.56.104)


---

## Scan Information

| Field | Details |
|-------|---------|
| Target IP | 192.168.56.104 |
| Hostname | metasploitable.localdomain |
| MAC Address | 08:00:27:CD:11:88 (Oracle VirtualBox) |
| Host Status | Up |
| Network | Host-Only (192.168.56.0/24) |
| Scanner | Nmap 7.99 on Kali Linux |
| Scan Date | Sun May 10, 2026 |

---

## Scans Performed

| Phase | Command | Duration | Output File |
|-------|---------|----------|-------------|
| Phase 1 — Full Port Discovery | `nmap -p- --min-rate 5000` | 39.43 seconds | open-ports.txt |
| Phase 2 — Deep Scan (version + scripts + OS) | `nmap -sV -sC -O -p [ports]` | 22.73 seconds | metasploitable-deep-scan.nmap |
| Phase 3 — Vulnerability Scripts | `nmap --script vuln` | 522.97 seconds | vuln-scan.txt |

---

## OS Detection

```
Device Type:      General Purpose
Running:          Linux 2.6.X
OS Details:       Linux 2.6.9 - 2.6.33
OS CPE:           cpe:/o:linux:linux_kernel:2.6
Network Distance: 1 hop
```

## Commands Used

```bash
#— Host Discovery
nmap -sn 192.168.56.0/24

## Host Discovery Results

```
```bash
Starting Nmap scan on 192.168.56.0/24

Nmap scan report for 192.168.56.1     — Host is up (VirtualBox gateway)
Nmap scan report for 192.168.56.101   — Host is up (Kali Linux — attacker)
Nmap scan report for 192.168.56.104   — Host is up (Metasploitable2 — target)

3 hosts found on subnet

## Scan Informat
```

---

## Open Ports — Full Discovery (Phase 1)

30 open TCP ports discovered out of 65535 scanned in 39.43 seconds.

| Port | Service | Port | Service |
|------|---------|------|---------|
| 21 | ftp | 1524 | ingreslock |
| 22 | ssh | 2049 | nfs |
| 23 | telnet | 2121 | ccproxy-ftp |
| 25 | smtp | 3306 | mysql |
| 53 | domain | 3632 | distccd |
| 80 | http | 5432 | postgresql |
| 111 | rpcbind | 5900 | vnc |
| 139 | netbios-ssn | 6000 | X11 |
| 445 | microsoft-ds | 6667 | irc |
| 512 | exec | 6697 | ircs-u |
| 513 | login | 8009 | ajp13 |
| 514 | shell | 8180 | http (Tomcat) |
| 1099 | rmiregistry | 8787 | msgsrvr |
| 32911 | RPC status | 54186 | mountd |
| 35581 | unknown | 55285 | nlockmgr |

---

## Service Version Detection (Phase 2)

| Port | Service | Version Detected |
|------|---------|-----------------|
| 21/tcp | FTP | vsftpd 2.3.4 |
| 22/tcp | SSH | OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0) |
| 23/tcp | Telnet | Linux telnetd |
| 25/tcp | SMTP | Postfix smtpd |
| 53/tcp | DNS | ISC BIND 9.4.2 |
| 80/tcp | HTTP | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| 111/tcp | RPC | rpcbind 2 |
| 139/tcp | NetBIOS | Samba smbd 3.X - 4.X |
| 445/tcp | SMB | Samba smbd 3.0.20-Debian |
| 512/tcp | exec | netkit-rsh rexecd |
| 513/tcp | login | OpenBSD or Solaris rlogind |
| 514/tcp | shell | Netkit rshd |
| 1099/tcp | Java RMI | GNU Classpath grmiregistry |
| 1524/tcp | bindshell | Metasploitable root shell |
| 2049/tcp | NFS | 2-4 (RPC #100003) |
| 2121/tcp | FTP | ProFTPD 1.3.1 |
| 3306/tcp | MySQL | MySQL 5.0.51a-3ubuntu5 |
| 3632/tcp | distccd | distccd v1 (GNU 4.2.4) |
| 5432/tcp | PostgreSQL | PostgreSQL DB 8.3.0 - 8.3.7 |
| 5900/tcp | VNC | VNC protocol 3.3 |
| 6000/tcp | X11 | access denied |
| 6667/tcp | IRC | UnrealIRCd |
| 6697/tcp | IRC | UnrealIRCd |
| 8009/tcp | AJP | Apache Jserv Protocol v1.3 |
| 8180/tcp | HTTP | Apache Tomcat/Coyote JSP 1.1 |
| 8787/tcp | Ruby DRb | Ruby DRb RMI (Ruby 1.8) |

---

## Additional Script Findings (Phase 2)

**FTP — Anonymous Login Confirmed:**
```
ftp-anon: Anonymous FTP login allowed (FTP code 230)
Logged in as: ftp
Control connection: plain text
Data connections: plain text
```

**SMTP — Expired SSL Certificate (16 years old):**
```
ssl-cert: commonName=ubuntu804-base.localdomain
Not valid before: 2010-03-17
Not valid after:  2010-04-16
SSLv2 supported (broken protocol)
Ciphers: SSL2_RC2, SSL2_RC4, SSL2_DES (all deprecated)
```

**SMB — Dangerous Configuration:**
```
Account used: guest
Authentication level: user
Message signing: DISABLED (dangerous, but default)
OS: Unix (Samba 3.0.20-Debian)
Computer name: metasploitable
FQDN: metasploitable.localdomain
```

**MySQL:**
```
Version: 5.0.51a-3ubuntu5
Protocol: 10
Status: Autocommit
Note: Accessible from network — no firewall restriction
```

---

## Vulnerability Scan Results (Phase 3)

### CRITICAL

---

**[CRITICAL-01] vsftpd 2.3.4 Backdoor — Port 21/tcp**

| Field | Detail |
|-------|--------|
| CVE | CVE-2011-2523 |
| BID | 48539 |
| CVSS | 10.0 |
| Disclosed | 2011-07-03 |
| Status | VULNERABLE — Actively Exploited |
| Metasploit | exploit/unix/ftp/vsftpd_234_backdoor |

What happened: vsftpd 2.3.4 was backdoored when an attacker compromised the official distribution server. Sending a username with `:)` triggers the backdoor and opens a root shell on port 6200.

**Nmap confirmed root access during scan:**
```
Shell command: id
Results: uid=0(root) gid=0(root)
```

This is the most critical finding — Nmap did not just detect this vulnerability, it actively exploited it and confirmed root-level command execution on the target system.

Remediation: Upgrade vsftpd to 2.3.5+. Disable FTP entirely and use SFTP over SSH.

---

**[CRITICAL-02] Root Bindshell — Port 1524/tcp**

| Field | Detail |
|-------|--------|
| CVE | N/A — Intentional backdoor |
| CVSS | 10.0 |
| Status | Open — No authentication |

What happened: Port 1524 has a root shell directly accessible with zero authentication. Nmap identified it as "Metasploitable root shell."

```bash
nc 192.168.56.104 1524
# Immediately returns: root@metasploitable:/#
```

Remediation: This port must be closed immediately. Its existence means the system is fully backdoored and must be considered completely compromised.

---

**[CRITICAL-03] Java RMI Registry RCE — Port 1099/tcp**

| Field | Detail |
|-------|--------|
| Status | VULNERABLE |
| Metasploit | exploit/multi/misc/java_rmi_server |

What happened: Default RMI registry configuration allows loading classes from attacker-controlled remote URLs, leading to remote code execution.

```
rmi-vuln-classloader: VULNERABLE
Default configuration of RMI registry allows loading classes from
remote URLs which can lead to remote code execution.
```

Remediation: Disable Java RMI if not needed. Restrict with Java security manager if required.

---

### HIGH

---

**[HIGH-01] SSL POODLE — Port 25/tcp and 5432/tcp**

| CVE | CVE-2014-3566 | BID | 70574 |
|-----|---------------|-----|-------|
| Disclosed | 2014-10-14 | Affected | SMTP + PostgreSQL |

SSLv3 nondeterministic CBC padding allows MitM attackers to recover plaintext via padding-oracle attack.

```
Check: TLS_RSA_WITH_AES_128_CBC_SHA — VULNERABLE
```

Remediation: Disable SSLv3. Configure TLS 1.2 or TLS 1.3 only.

---

**[HIGH-02] Logjam TLS Downgrade — Port 25/tcp**

| CVE | CVE-2015-4000 | BID | 74733 |
|-----|---------------|-----|-------|
| Disclosed | 2015-05-19 | |  |

DHE_EXPORT ciphers allow MitM downgrade to 512-bit export-grade crypto.

```
Cipher Suite: TLS_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA
Modulus Length: 512 bits (trivially breakable)
```

Remediation: Disable DHE_EXPORT ciphers. Use DH groups of 2048 bits minimum.

---

**[HIGH-03] Anonymous Diffie-Hellman MitM — Port 25/tcp**

Anonymous DH provides no authentication — fully vulnerable to active MitM.

```
Cipher Suite: TLS_DH_anon_WITH_AES_256_CBC_SHA
Modulus Length: 1024 bits
```

Remediation: Disable all anonymous cipher suites.

---

**[HIGH-04] Weak DH Group Strength — Port 25/tcp and 5432/tcp**

1024-bit DH groups are insufficient and may be broken by advanced adversaries.

```
Port 25:   TLS_DHE_RSA_WITH_DES_CBC_SHA — 1024 bit
Port 5432: TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA — 1024 bit
```

Remediation: Use 2048-bit DH groups minimum. Prefer ECDHE suites.

---

**[HIGH-05] CCS Injection (OpenSSL MitM) — Port 5432/tcp**

| CVE | CVE-2014-0224 | Risk | High |
|-----|---------------|------|------|

OpenSSL improperly handles ChangeCipherSpec, allowing MitM to force zero-length master key and hijack sessions.

Remediation: Upgrade OpenSSL to 0.9.8za / 1.0.0m / 1.0.1h or later.

---

### MEDIUM

---

**[MEDIUM-01] HTTP TRACE Enabled — Port 80/tcp**

```
http-trace: TRACE is enabled
```

Enables Cross-Site Tracing (XST) attacks to steal cookies. Remediation: `TraceEnable off` in Apache config.

---

**[MEDIUM-02] CSRF Vulnerabilities — Port 80/tcp**

Forms without CSRF tokens found at:
```
/dvwa/             Form action: login.php
/dvwa/login.php    Form action: login.php
/mutillidae/       Form action: ./index.php?page=user-info.php
```

Remediation: Implement CSRF tokens on all state-changing forms.

---

**[MEDIUM-03] Sensitive Paths Exposed — Port 80/tcp**

```
http-enum findings:
  /tikiwiki/      Tikiwiki CMS
  /phpinfo.php    PHP configuration disclosure
  /phpMyAdmin/    Database admin interface
  /doc/           Directory listing enabled
  /icons/         Directory listing enabled
  /test/          Test page
```

`phpinfo.php` reveals server internals. `/phpMyAdmin/` exposes direct database access. Remediation: Remove phpinfo.php, restrict phpMyAdmin, disable directory listing.

---

**[MEDIUM-04] SMB Signing Disabled — Port 445/tcp**

```
message_signing: disabled (dangerous, but default)
```

Allows SMB relay attacks. Remediation: Enable SMB signing in Samba config.

---

**[MEDIUM-05] Telnet Running — Port 23/tcp**

Plaintext protocol — credentials visible in any packet capture. Remediation: Disable Telnet, use SSH only.

---

**[MEDIUM-06] Anonymous FTP Login — Port 21/tcp**

```
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Remediation: Disable anonymous FTP. Use SFTP instead.

---

### INFORMATIONAL

| # | Finding | Detail |
|---|---------|--------|
| INFO-01 | SSLv2 supported on port 25 | Broken protocol, weak ciphers (RC2, RC4, DES) |
| INFO-02 | Expired SSL cert on ports 25 and 5432 | Expired April 2010 — 16 years ago |
| INFO-03 | distccd exposed on port 3632 | CVE-2004-2687 — RCE risk, should not be network-facing |
| INFO-04 | MS10-054 and MS10-061 | Checked — NOT vulnerable |

---

## Risk Summary

| Severity | Count |
|----------|-------|
| Critical | 3 |
| High | 5 |
| Medium | 6 |
| Informational | 4 |
| **Total** | **18** |

---

## Remediation Priority

**Immediate:**
- Close port 1524 (root bindshell — system fully compromised)
- Remove/upgrade vsftpd 2.3.4 (root access confirmed)
- Disable Java RMI (port 1099)
- Disable Telnet (port 23)

**This Week:**
- Disable SSLv2 and SSLv3 on all services
- Remove DHE_EXPORT and anonymous DH ciphers
- Upgrade OpenSSL (CCS Injection fix)
- Use 2048-bit DH groups minimum
- Disable anonymous FTP

**This Month:**
- Disable HTTP TRACE
- Add CSRF tokens to web forms
- Remove phpinfo.php, restrict phpMyAdmin
- Enable SMB message signing
- Replace expired SSL certificates

---

## Conclusion

The target machine (192.168.56.104 — Metasploitable2) is critically vulnerable with 3 unauthenticated remote root access vectors confirmed. The vsftpd backdoor was actively exploited during scanning, returning `uid=0(root) gid=0(root)`. Port 1524 provides a direct root shell with no authentication. Combined with 5 high-severity TLS weaknesses and 6 medium findings, this system must be treated as fully compromised and isolated from any real network immediately.

Scan conducted in isolated lab environment (Host-Only 192.168.56.0/24) for educational purposes — ApexPlanet Cybersecurity Internship Task 2.

---

## Screenshots Index

| File | Description |
|------|-------------|
| `screenshots/phase1-open-ports.png` | 30 open ports from full scan |
| `screenshots/phase2-version-detection.png` | Service versions and OS output |
| `screenshots/phase3-vuln-scan.png` | Vulnerability script results |
| `screenshots/vsftpd-root-uid0.png` | uid=0(root) confirmed |
| `screenshots/bindshell-port-1524.png` | Netcat to port 1524 — root shell |

---

*Report by: Professorshubhx | ApexPlanet Internship 
