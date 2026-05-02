# Lab Setup Report
### ApexPlanet Internship — Task 1 Deliverable
**Author:** Professorshubhx 

---

## Overview

This report documents the complete setup of a professional cybersecurity lab environment built as part of Task 1 of the ApexPlanet Software Pvt. Ltd. Cybersecurity Internship. The lab consists of isolated virtual machines configured for safe penetration testing practice, with no exposure to external networks.

---

## Lab Architecture

```
Host Machine
│
├── Hypervisor: VirtualBox
│
├── VM 1: Kali Linux (Attacker Machine)
│         OS: Kali Linux Rolling
│         RAM: 2GB+
│         Network: Host-Only Adapter (192.168.56.x)
│         Role: Offensive operations, tool usage, recon
│
├── VM 2: Metasploitable2 (Target Machine)
│         OS: Ubuntu 8.04 (intentionally vulnerable)
│         RAM: 512MB
│         Network: Host-Only Adapter (192.168.56.x)
│         Role: Vulnerable target for practice
│
└── Network: Host-Only (192.168.56.0/24)
            Isolated from internet and host network
            VMs communicate with each other only
```

---

## Step 1 — Hypervisor Installation

**Software:** VirtualBox (free, open source)
**Download:** https://www.virtualbox.org/

VirtualBox was installed on the host machine. The VirtualBox Extension Pack was also installed to enable USB support and improved VM performance.

After installation, a Host-Only network was created:
- VirtualBox → File → Host Network Manager → Create
- Network: `192.168.56.0/24`
- DHCP enabled: `192.168.56.100` to `192.168.56.254`

---

## Step 2 — Kali Linux Setup

**Source:** Pre-built VirtualBox OVA from https://www.kali.org/get-kali/#kali-virtual-machines

**Import Process:**
1. VirtualBox → File → Import Appliance
2. Selected the downloaded `.ova` file
3. Kept default settings, clicked Import
4. Assigned Host-Only adapter in VM Network settings

**Post-install configuration:**

```bash
# System update
sudo apt update && sudo apt upgrade -y

# Verified network assignment
ip a
# Kali IP confirmed: 192.168.56.xxx

# Verified tools available
which nmap wireshark burpsuite netcat
```

**Screenshot:** `screenshots/kali-desktop.png`
**Screenshot:** `screenshots/kali-ip-address.png`

---

## Step 3 — Metasploitable2 Setup

**Source:** https://sourceforge.net/projects/metasploitable/

Metasploitable2 is an intentionally vulnerable Linux VM designed for security training. It contains known vulnerabilities across multiple services including FTP, SSH, HTTP, MySQL, Samba, and more.

**Import Process:**
1. Created new VM in VirtualBox (Linux, Ubuntu 32-bit)
2. Attached Metasploitable2's `.vmdk` as the storage disk
3. Set RAM to 512MB
4. Assigned Host-Only adapter

**Verification:**

```bash
# Logged in with default credentials: msfadmin / msfadmin
# Ran ip a to confirm IP assignment
ip a
# Metasploitable2 IP confirmed: 192.168.56.xxx
```

**Screenshot:** `screenshots/metasploitable-login.png`
**Screenshot:** `screenshots/metasploitable-ip.png`

---

## Step 4 — Network Configuration Verification

Both VMs placed on the same Host-Only network (`192.168.56.0/24`), isolated from the internet.

**Connectivity test from Kali to Metasploitable2:**

```bash
ping -c 4 192.168.56.101
```

```
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=0.487 ms
64 bytes from 192.168.56.101: icmp_seq=2 ttl=64 time=0.412 ms
64 bytes from 192.168.56.101: icmp_seq=3 ttl=64 time=0.398 ms
64 bytes from 192.168.56.101: icmp_seq=4 ttl=64 time=0.421 ms

--- 192.168.56.101 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

Result: VMs can communicate on the isolated Host-Only network.

**Screenshot:** `screenshots/ping-kali-to-metasploitable.png`

---

## Step 5 — Nmap Scan Verification

An Nmap scan was run from Kali against Metasploitable2 to confirm services are discoverable and the lab is fully functional.

**Command:**

```bash
nmap -sV -sC 192.168.56.101
```

**Key findings:**

```
PORT     STATE  SERVICE     VERSION
21/tcp   open   ftp         vsftpd 2.3.4
22/tcp   open   ssh         OpenSSH 4.7p1 Debian
23/tcp   open   telnet      Linux telnetd
25/tcp   open   smtp        Postfix smtpd
53/tcp   open   domain      ISC BIND 9.4.2
80/tcp   open   http        Apache httpd 2.2.8
139/tcp  open   netbios-ssn Samba smbd 3.X
445/tcp  open   netbios-ssn Samba smbd 3.0.20
3306/tcp open   mysql       MySQL 5.0.51a
5432/tcp open   postgresql  PostgreSQL DB 8.3.0
8180/tcp open   http        Apache Tomcat 5.5
```

This confirms Metasploitable2 is running intentionally vulnerable versions of multiple services — the lab is working as intended.

**Screenshot:** `screenshots/nmap-scan-metasploitable.png`

---

## Step 6 — Wireshark Test Capture

Wireshark was launched on Kali and a test packet capture was performed to verify traffic capture functionality.

**Steps:**
1. Opened Wireshark, selected Host-Only network interface
2. Started capture
3. Ran `ping -c 4 192.168.56.101` in terminal
4. Stopped capture
5. Applied filter: `icmp`

ICMP Echo Request and Echo Reply packets were captured and visible, confirming Wireshark is capturing lab traffic correctly.

**Screenshot:** `screenshots/wireshark-capture.png`

---

## Step 7 — DVWA Access Verification

DVWA (Damn Vulnerable Web App) comes pre-installed on Metasploitable2. Access was verified through Kali's browser.

**URL:** `http://192.168.56.101/dvwa`
**Default credentials:** admin / password

DVWA loaded successfully. Security level set to Low for practice exercises.

**Screenshot:** `screenshots/dvwa-loaded.png`

---

## Lab Summary

| Component | Status | Details |
|-----------|--------|---------|
| VirtualBox | Installed | With Extension Pack |
| Kali Linux | Running | Host-Only network, updated |
| Metasploitable2 | Running | Host-Only network |
| Host-Only Network | Configured | 192.168.56.0/24, DHCP enabled |
| Kali → Metasploitable connectivity | Verified | 0% packet loss |
| Nmap scan | Completed | 11+ open services discovered |
| Wireshark capture | Working | ICMP traffic captured |
| DVWA | Accessible | Running on Metasploitable2 |

All components are operational. The lab is isolated, functional, and ready for security practice.

---

## Screenshots Index

| File | Description |
|------|-------------|
| `screenshots/kali-desktop.png` | Kali Linux desktop after login |
| `screenshots/kali-ip-address.png` | `ip a` output showing Kali's Host-Only IP |
| `screenshots/metasploitable-login.png` | Metasploitable2 terminal login |
| `screenshots/metasploitable-ip.png` | `ip a` output on Metasploitable2 |
| `screenshots/ping-kali-to-metasploitable.png` | Successful ping from Kali to Metasploitable2 |
| `screenshots/nmap-scan-metasploitable.png` | Nmap -sV scan results |
| `screenshots/wireshark-capture.png` | Wireshark showing ICMP packets |
| `screenshots/dvwa-loaded.png` | DVWA running in Kali browser |

---

## Security Notes

- Metasploitable2 is **never** connected to the internet or home network
- All testing is confined to the Host-Only network (`192.168.56.0/24`)
- Default credentials on both VMs were noted and would be changed in any real deployment
- The lab is torn down (VMs powered off) when not in use

---

*Report prepared by: Professorshubhx*
*Internship: ApexPlanet Software Pvt. Ltd.*
