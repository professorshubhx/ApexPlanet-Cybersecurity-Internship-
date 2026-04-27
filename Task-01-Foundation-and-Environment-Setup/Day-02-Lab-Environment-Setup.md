# Day 02 — Lab Environment Setup
### ApexPlanet Internship | Task 1: Foundation & Environment Setup
**Timeline:** Days 1–12 | **Author:** Professorshubhx

---

## What We're Doing Today

Before you can practice any attack or defense, you need a safe place to do it. That's what a lab environment is — an isolated setup where you can break things, test tools, and simulate attacks without affecting any real system or network.

Today we build that lab from scratch.

---

## Why a Separate Lab?

Running hacking tools on your real machine or your home network is a bad idea — not because the tools are illegal (they aren't, in a lab context), but because you could accidentally affect real systems, get flagged by your ISP, or corrupt your own machine. A virtual lab keeps everything contained.

The setup we're building today is exactly what junior SOC analysts and ethical hackers use when learning. It's industry-standard.

---

## The Lab Architecture

Here's how it looks when fully set up:

```
Your Physical Machine (Host)
        |
        | runs
        |
  [VirtualBox / VMware]
        |
        |---> Kali Linux VM       (Attacker Machine)
        |---> Metasploitable2 VM  (Target Machine - intentionally vulnerable)
        |---> DVWA VM             (Web App Target - optional but recommended)
        |
  [Host-Only Network Adapter]
        |
  All VMs talk to each other, isolated from the internet
```

The key thing here is **Host-Only Network**. This means the VMs can communicate with each other but are cut off from the internet. So even if you run aggressive tools like Nmap or Metasploit, nothing leaves your machine.

---

## Step 1 — Install a Hypervisor (VirtualBox or VMware)

A hypervisor is software that lets you run multiple operating systems inside your main OS. Think of it as an OS inside an OS.

### Option A: VirtualBox (Free, Recommended for Beginners)

Download from: https://www.virtualbox.org/

- Works on Windows, macOS, Linux
- Completely free
- Good enough for all beginner-to-intermediate labs

After installing, also install the **VirtualBox Extension Pack** — it adds USB support and better performance.

### Option B: VMware Workstation Player (Free version)

Download from: https://www.vmware.com/products/workstation-player.html

- Slightly better performance than VirtualBox
- Free version has limitations but is fine for this lab

**Which one to pick?** If you're on Windows, either works. VMware is slightly smoother; VirtualBox is totally free with no strings attached. For this internship, VirtualBox is what most people use.

---

## Step 2 — Install Kali Linux (Attacker Machine)

Kali Linux is a Debian-based Linux distribution made specifically for penetration testing. It comes pre-installed with 600+ security tools — Nmap, Wireshark, Metasploit, Burp Suite, Aircrack-ng, and more.

### Getting Kali

Download the pre-built VirtualBox image from: https://www.kali.org/get-kali/#kali-virtual-machines

Don't install from ISO if you're just starting out — the pre-built VM image is faster and already configured.

### Importing into VirtualBox

1. Open VirtualBox
2. Click File → Import Appliance
3. Select the downloaded `.ova` file
4. Keep default settings, click Import
5. Once imported, click Start

**Default credentials:**
- Username: `kali`
- Password: `kali`

Change these immediately after first login.

### First things to do after booting Kali

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Check your IP address (note this down)
ip a

# Verify internet works (before setting up Host-Only later)
ping google.com -c 4
```

---

## Step 3 — Install Metasploitable2 (Target Machine)

Metasploitable2 is an intentionally vulnerable Linux VM. It's literally designed to be hacked. Every service on it has known vulnerabilities — FTP, SSH, HTTP, MySQL, Samba, and more.

Download from: https://sourceforge.net/projects/metasploitable/

### Importing into VirtualBox

1. Metasploitable2 comes as a `.vmdk` file (VMware disk format)
2. In VirtualBox: New → Linux → Ubuntu 32-bit
3. When asked about storage: "Use an existing virtual hard disk file" → select the `.vmdk`
4. Set RAM to 512MB (it doesn't need much)

**Default credentials:**
- Username: `msfadmin`
- Password: `msfadmin`

**Important:** Never expose Metasploitable2 to the internet. It's full of open vulnerabilities. Keep it Host-Only always.

---

## Step 4 — Install DVWA (Optional but Recommended)

DVWA stands for Damn Vulnerable Web Application. It's a web app built to be hacked — it has SQL injection, XSS, CSRF, file inclusion, and other OWASP Top 10 vulnerabilities baked in.

You can either install it on Metasploitable2 (it comes pre-installed there) or set it up separately using XAMPP.

If you want to run it standalone:

```bash
# On Kali or any Linux machine with Apache + PHP + MySQL (LAMP stack)
git clone https://github.com/digininja/DVWA.git
# Then move to /var/www/html and configure config.inc.php
```

For now, just use the DVWA that's already on Metasploitable2 — it's accessible at `http://<metasploitable-ip>/dvwa`

---

## Step 5 — Configure the Host-Only Network

This is the most important step. Without this, your VMs are either connected to the internet (unsafe for testing) or can't talk to each other (useless for attack/defense practice).

### In VirtualBox

**Create the Host-Only Network:**
1. File → Host Network Manager → Create
2. Note the IP range created (usually `192.168.56.0/24`)
3. Enable DHCP server

**Assign it to each VM:**
1. Select VM → Settings → Network
2. Adapter 1: Change to "Host-only Adapter"
3. Select the network you just created
4. Repeat for both Kali and Metasploitable2

### Verify the connection

Boot both VMs, then from Kali:

```bash
# Find Metasploitable2's IP
ip a                          # Check Kali's IP first
nmap -sn 192.168.56.0/24     # Scan the host-only subnet for live hosts

# Ping Metasploitable2 (replace with actual IP)
ping 192.168.56.101 -c 4

# Quick port scan to confirm Metasploitable is reachable
nmap -sV 192.168.56.101
```

If you see Metasploitable2 respond and ports like 21 (FTP), 22 (SSH), 80 (HTTP) open — your lab is working.

---

## Key Linux Commands for Lab Navigation

Since you'll be spending a lot of time in the terminal, here are the essential commands for navigating the lab:

```bash
# Networking
ip a                          # Show IP addresses of all interfaces
ip route                      # Show routing table
ping <ip> -c 4               # Send 4 ICMP pings to check connectivity
netstat -tuln                 # Show open ports and listening services
ss -tuln                      # Modern alternative to netstat

# System info
uname -a                      # Kernel version and system info
whoami                        # Current user
id                            # User ID and group memberships
hostname                      # Machine hostname

# File navigation
ls -la                        # List all files with permissions
pwd                           # Print working directory
cd /path/to/dir               # Change directory
cat filename                  # Read file contents
less filename                 # Read file with scrolling

# Services (useful for checking what's running)
systemctl status apache2      # Check if Apache web server is running
service --status-all          # List all services
ps aux                        # Show all running processes
```

---

## Verify Your Lab is Working

Before moving to Day 3, confirm these things are working:

**Checklist:**

- [ ] VirtualBox (or VMware) installed and opens without errors
- [ ] Kali Linux boots and you can log in
- [ ] Metasploitable2 boots and you can log in
- [ ] Both VMs are on Host-Only network
- [ ] Kali can ping Metasploitable2 successfully
- [ ] Nmap scan from Kali shows Metasploitable2's open ports
- [ ] You can access DVWA at `http://<metasploitable-ip>/dvwa` from Kali's browser

If all of these work, your lab is ready.

---

## Common Issues and Fixes

**VirtualBox won't start a VM (Kernel driver not installed)**

```bash
# On Linux host:
sudo modprobe vboxdrv
# If that fails, reinstall VirtualBox kernel modules
sudo apt install --reinstall virtualbox-dkms
```

**VMs can't ping each other**

- Make sure both VMs are using the SAME Host-Only adapter
- Check that the Host-Only DHCP is enabled in VirtualBox Host Network Manager
- Temporarily disable firewalls on both VMs to test: `sudo ufw disable`

**Kali runs very slowly**

- Increase RAM in VM settings (minimum 2GB recommended for Kali)
- Enable VT-x/AMD-V (virtualization) in your BIOS if not already on
- Enable 3D acceleration in Display settings

**Metasploitable2 shows a black screen**

- It's a headless VM, so the login prompt may take 30–60 seconds
- Press Enter if you see a blank screen — the login: prompt is there

---

## What You Just Built

You now have a personal penetration testing lab that mirrors what professionals use in the real world. Kali is your offensive machine. Metasploitable2 is your target. DVWA is your web application target. The Host-Only network means nothing escapes.

From Day 3 onward, you'll start using this lab actively — Linux fundamentals, then networking recon, then actual tool usage.

---

## Deliverable Reminder (Task 1)

For the lab setup report, take screenshots of:
- Kali Linux desktop (after login)
- Metasploitable2 terminal (showing `ip a` output)
- Successful ping from Kali to Metasploitable2
- Nmap scan output showing Metasploitable2's open ports
- DVWA loaded in Kali's browser

Save them in your repo at `Task-01-Foundation-and-Environment-Setup/screenshots/`

---

## Resources

- Kali Linux Documentation: https://www.kali.org/docs/
- Metasploitable2 Guide: https://docs.rapid7.com/metasploit/metasploitable-2/
- VirtualBox Manual: https://www.virtualbox.org/manual/
- DVWA GitHub: https://github.com/digininja/DVWA

---

## Connect

- YouTube: [@professorshubhx](https://youtube.com/@Professorshubhx)
- LinkedIn: [Shubham Chaurasiya](https://linkedin.com/in/shubham-chaurasiya-60932a359/)


---

*Day 02 Complete | Next: Day 03 — Linux Fundamentals*
