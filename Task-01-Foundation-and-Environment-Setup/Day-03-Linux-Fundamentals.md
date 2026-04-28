# Day 03 — Linux Fundamentals
### ApexPlanet Internship | Task 1: Foundation & Environment Setup
**Timeline:** Days 1–12 | **Author:** Professorshubhx

---

## Why Linux for Cybersecurity

Almost every server on the internet runs Linux. Most hacking tools are built for Linux. Kali Linux, the industry-standard penetration testing OS, is Linux. If you're going into cybersecurity — SOC analyst, pentester, bug bounty hunter, anything — you need to be comfortable in the Linux terminal.

The good news: you don't need to memorize everything. You need to understand what's possible and know how to look things up. That's what today is about.

All commands below should be practiced on your Kali Linux VM from Day 02.

---

## 1. File System Navigation

Linux has a single unified file system tree starting from `/` (called root). Everything — files, devices, processes, network interfaces — is represented as a file somewhere in this tree.

### The Directory Structure

```
/                    Root of the entire file system
├── bin/             Essential binaries (ls, cp, mv, cat...)
├── etc/             System configuration files
├── home/            User home directories (/home/kali for the kali user)
├── var/             Variable data (logs, databases, mail)
│   └── log/         System and application logs (important for SOC work)
├── tmp/             Temporary files (cleared on reboot)
├── usr/             User programs and utilities
├── root/            Home directory for the root user
├── proc/            Virtual filesystem — live kernel and process info
├── dev/             Device files (hard drives, USBs, terminals)
└── opt/             Optional/third-party software
```

As a SOC analyst, `/var/log/` is where you'll spend a lot of time — that's where system logs live.

### Navigation Commands

```bash
pwd                  # Print Working Directory — tells you where you are
ls                   # List files in current directory
ls -l                # Long format (shows permissions, size, owner, date)
ls -a                # Show hidden files (files starting with .)
ls -la               # Both combined — most useful form
cd /path/to/dir      # Change to that directory
cd ..                # Go one level up
cd ~                 # Go to your home directory
cd -                 # Go back to previous directory

# Examples
cd /var/log          # Navigate to log directory
ls -la               # See all logs with details
cd ~                 # Go back home
```

### Viewing Files

```bash
cat filename         # Print entire file to terminal
less filename        # Scroll through file (q to quit)
head filename        # Show first 10 lines
head -n 20 filename  # Show first 20 lines
tail filename        # Show last 10 lines
tail -f filename     # Follow file in real-time (great for live logs)
grep "keyword" file  # Search for keyword inside a file
```

The `tail -f` command is something you'll use as a SOC analyst to watch logs update in real-time during an incident.

---

## 2. File and Directory Operations

```bash
# Creating
touch newfile.txt        # Create an empty file
mkdir myfolder           # Create a new directory
mkdir -p a/b/c           # Create nested directories all at once

# Copying and Moving
cp file1 file2           # Copy file1 to file2
cp -r folder1 folder2    # Copy entire folder (recursive)
mv file1 /path/to/dest   # Move file (also used to rename)
mv oldname.txt new.txt   # Rename a file

# Deleting
rm file.txt              # Delete a file
rm -r folder/            # Delete a folder and everything inside it
rm -rf folder/           # Force delete without confirmation (use carefully)

# Finding files
find / -name "passwd"           # Find file named passwd anywhere on system
find /home -name "*.txt"        # Find all .txt files in /home
find /var/log -mtime -1         # Find files modified in last 24 hours
locate filename                 # Faster search using indexed database
```

---

## 3. File and Directory Permissions

This is one of the most important topics in Linux — and one that beginners skip too fast. Permissions determine who can read, write, or execute every single file on the system. Understanding this is critical for both offense (finding misconfigured permissions) and defense (locking things down properly).

### Reading Permission Output

When you run `ls -la`, you see something like this:

```
-rwxr-xr--  1  kali  kali  4096  Apr 25  script.sh
drwxr-xr-x  2  root  root  4096  Apr 25  logs/
```

Let's break down `-rwxr-xr--`:

```
Position 1:   - = regular file | d = directory | l = symbolic link
Positions 2-4: rwx = Owner permissions (read, write, execute)
Positions 5-7: r-x = Group permissions (read, no write, execute)
Positions 8-10: r-- = Others permissions (read only)
```

So `-rwxr-xr--` means:
- It's a regular file
- Owner can read, write, and execute
- Group can read and execute, not write
- Everyone else can only read

### Permission Values (Numeric / Octal)

Each permission has a numeric value:

```
r = 4
w = 2
x = 1
- = 0
```

You add them up for each group:

```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0
```

So `rwxr-xr--` = `754`

### chmod — Change Permissions

```bash
chmod 755 script.sh      # Owner: rwx, Group: r-x, Others: r-x
chmod 644 notes.txt      # Owner: rw-, Group: r--, Others: r--
chmod 600 id_rsa         # Owner: rw-, Group: none, Others: none (SSH key standard)
chmod 777 file           # Everyone can do everything (avoid this in production)

# Symbolic method (easier to read)
chmod u+x script.sh      # Add execute for user (owner)
chmod g-w file.txt       # Remove write from group
chmod o+r file.txt       # Add read for others
chmod a+x script.sh      # Add execute for all (user, group, others)
```

Common permission patterns to memorize:

```
755  — Standard for executables and directories (owner full, others read+execute)
644  — Standard for text files (owner read+write, others read only)
600  — Private files like SSH keys (owner read+write only, nobody else)
700  — Private directories (owner only)
```

### chown — Change Ownership

```bash
chown kali file.txt              # Change file owner to kali
chown kali:kali file.txt         # Change owner and group to kali
chown -R kali /home/kali/folder  # Recursively change ownership of entire folder
chown root:root /etc/shadow      # This is how shadow file should be owned
```

### Practical Security Example

Finding world-writable files (a common misconfiguration attackers look for):

```bash
find / -perm -o+w -type f 2>/dev/null     # Find files anyone can write to
find / -perm -4000 -type f 2>/dev/null    # Find SUID files (run as owner, not caller)
```

SUID files are particularly interesting in penetration testing because if a SUID binary has a vulnerability, you can use it to escalate privileges.

---

## 4. Package Management

On Kali Linux (Debian-based), you use `apt` to install, update, and remove software.

### apt — Advanced Package Tool

```bash
# Updating
sudo apt update                  # Refresh the package list (always do this first)
sudo apt upgrade                 # Install available updates
sudo apt update && sudo apt upgrade -y   # Do both in one command

# Installing
sudo apt install nmap            # Install nmap
sudo apt install wireshark       # Install Wireshark
sudo apt install -y packagename  # Install without asking for confirmation

# Removing
sudo apt remove packagename      # Remove package (keeps config files)
sudo apt purge packagename       # Remove package AND config files
sudo apt autoremove              # Remove unused dependencies

# Searching
apt search keyword               # Search for a package by keyword
apt show packagename             # Show details about a package

# Listing
dpkg -l                          # List all installed packages
dpkg -l | grep nmap              # Check if nmap is installed
```

### dpkg — Low-Level Package Manager

`dpkg` is what `apt` uses under the hood. You interact with it directly when installing `.deb` files (downloaded packages).

```bash
dpkg -i package.deb              # Install a .deb file
dpkg -r packagename              # Remove a package
dpkg -l                          # List installed packages
dpkg -L packagename              # List files installed by a package
dpkg --get-selections            # Show all packages and their status
```

### Installing Tools Not in apt

Sometimes a tool isn't in the apt repository. You install it from GitHub:

```bash
git clone https://github.com/user/tool.git
cd tool
pip install -r requirements.txt   # If Python-based
# OR
make && make install               # If C-based
```

---

## 5. Networking Commands

These are the commands you'll use constantly — both in the lab and in a real SOC environment for investigating alerts.

### Basic Network Info

```bash
ip a                             # Show all network interfaces and IP addresses
ip a show eth0                   # Show info for specific interface (eth0)
ip route                         # Show routing table
ip route show default            # Show default gateway

# Older alternative (still works on many systems)
ifconfig                         # Show network interfaces
ifconfig eth0                    # Specific interface
```

### Connectivity Testing

```bash
ping 8.8.8.8                     # Ping Google DNS (tests internet connectivity)
ping -c 4 192.168.56.101         # Send exactly 4 pings to target
ping -i 0.2 target               # Ping every 0.2 seconds (faster)

traceroute 8.8.8.8               # Show every hop between you and destination
traceroute -n 8.8.8.8            # Same but show IPs instead of hostnames (faster)

# Windows equivalent for reference
# tracert 8.8.8.8
```

`traceroute` is useful for understanding network paths and identifying where traffic is being blocked or rerouted.

### Open Ports and Connections

```bash
netstat -tuln                    # Show all listening ports (TCP and UDP)
netstat -tulnp                   # Same + show which process owns each port
netstat -an                      # Show all connections (listening + established)

ss -tuln                         # Modern replacement for netstat (faster)
ss -tulnp                        # With process info

# Check what's on a specific port
sudo lsof -i :80                 # What process is using port 80?
sudo lsof -i :443                # What process is using port 443?
```

### DNS Queries

```bash
nslookup google.com              # Query DNS for google.com
dig google.com                   # More detailed DNS query
dig google.com MX                # Query mail records
dig google.com ANY               # Query all record types
dig @8.8.8.8 google.com          # Query using specific DNS server (Google's)

host google.com                  # Simple hostname to IP lookup
```

### Practical Networking Commands for the Lab

```bash
# Discover live hosts on your lab network
nmap -sn 192.168.56.0/24

# Check what services are running on Metasploitable2
nmap -sV 192.168.56.101

# Watch network traffic in real-time (requires root)
sudo tcpdump -i eth0
sudo tcpdump -i eth0 host 192.168.56.101   # Filter by host
sudo tcpdump -i eth0 port 80               # Filter by port
sudo tcpdump -i eth0 -w capture.pcap       # Save to file for Wireshark analysis
```

---

## 6. Users, Groups, and Privilege Escalation

Understanding users and sudo is critical for both using Linux and understanding how privilege escalation attacks work.

```bash
whoami                           # Show current username
id                               # Show user ID, group ID, and group memberships
who                              # Show who is logged in
w                                # Show logged-in users and what they're doing
last                             # Show login history

# Switching users
su username                      # Switch to another user (needs their password)
su -                             # Switch to root (needs root password)
sudo command                     # Run a single command as root
sudo -i                          # Open a root shell

# User management
cat /etc/passwd                  # List all users on the system
cat /etc/shadow                  # Password hashes (root only)
cat /etc/group                   # List all groups

# Adding/removing users (root required)
sudo useradd -m newuser          # Create new user with home directory
sudo passwd newuser              # Set password for user
sudo userdel -r newuser          # Delete user and their home directory
```

### The sudo File

```bash
sudo cat /etc/sudoers            # Who can run what as root
sudo visudo                      # Safely edit the sudoers file
```

A common privilege escalation technique in CTFs and real engagements is finding a user who has been given unnecessary sudo permissions:

```bash
sudo -l                          # List what you can run as sudo (as current user)
```

If the output shows something like `(ALL) NOPASSWD: /usr/bin/vim`, that's exploitable.

---

## 7. Processes

```bash
ps aux                           # Show all running processes with details
ps aux | grep apache             # Find apache process specifically
top                              # Live process monitor (like Task Manager)
htop                             # Better version of top (install with apt)

kill 1234                        # Kill process by PID (process ID)
kill -9 1234                     # Force kill (SIGKILL) — process can't ignore this
killall apache2                  # Kill all processes named apache2

# Background processes
command &                        # Run command in background
jobs                             # List background jobs
fg 1                             # Bring job 1 to foreground
```

---

## 8. Text Processing (Essential for Log Analysis)

As a SOC analyst, you'll often be parsing logs. These commands are how you do it without opening a GUI.

```bash
grep "error" /var/log/syslog              # Find lines containing "error"
grep -i "failed" /var/log/auth.log        # Case-insensitive search
grep -n "root" /etc/passwd               # Show line numbers
grep -v "comment" file                    # Show lines NOT containing "comment"
grep -r "password" /etc/                 # Recursive search through directory

# Combining commands with pipes (|)
cat /var/log/auth.log | grep "Failed"    # Find failed login attempts
cat /var/log/auth.log | grep "Failed" | wc -l   # Count them

# Sorting and uniqueness
sort file.txt                            # Sort alphabetically
sort -n file.txt                         # Sort numerically
sort | uniq                              # Remove duplicate lines
sort | uniq -c                           # Count occurrences of each unique line

# Cut specific fields
cut -d: -f1 /etc/passwd                  # Extract usernames (field 1, colon-separated)

# AWK for column-based processing
awk '{print $1}' file.txt                # Print first column
awk -F: '{print $1}' /etc/passwd         # Print usernames from passwd file

# Real log analysis example
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn
# This shows the top IP addresses with failed SSH login attempts
```

---

## Quick Reference Cheatsheet

```
NAVIGATION          PERMISSIONS         NETWORKING
pwd                 chmod 755 file      ip a
ls -la              chmod 644 file      ping host
cd /path            chown user file     traceroute host
cat file            find / -perm -4000  netstat -tuln
tail -f file                            nmap -sV host

PACKAGES            USERS               PROCESSES
apt update          whoami              ps aux
apt install pkg     sudo -l             top
dpkg -i pkg.deb     id                  kill -9 PID
apt remove pkg      cat /etc/passwd     jobs
```

---

## Practice Tasks

Do these on your Kali VM before moving to Day 04:

1. Navigate to `/var/log` and use `ls -la` — note the permissions on the log files
2. Create a directory called `lab-notes` in your home folder
3. Create a file inside it, write some text, and read it back with `cat`
4. Change the file permissions to `600` using `chmod`
5. Run `ip a` and note Kali's IP address on the Host-Only network
6. Ping your Metasploitable2 VM from Kali
7. Run `netstat -tuln` and identify which ports are open on Kali
8. Use `grep "Failed" /var/log/auth.log` — if empty, try SSHing with wrong credentials first to generate some logs
9. Run `ps aux` and find the PID of the bash process
10. Use `sudo -l` and see what commands you're allowed to run as sudo

---

## Resources

- Linux Command Reference: https://linux.die.net/
- OverTheWire Bandit (Linux practice wargame): https://overthewire.org/wargames/bandit/
- Explainshell (paste any command, get explanation): https://explainshell.com/


---

## Connect

- Tryhackme: [@professorshubhx](https://tryhackme.com/p/professorshubhx)
- LinkedIn: [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)

---

*Day 03 Complete | Next: Day 04 — Networking Basics (OSI, TCP/IP, DNS, Subnetting)*
