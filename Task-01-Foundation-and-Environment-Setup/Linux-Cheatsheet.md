# Linux Cheatsheet
### ApexPlanet Internship — Task 1 Deliverable
**Author:** Professorshubhx | Defronix Cybersecurity Academy

---

## Navigation

```bash
pwd                        # Where am I?
ls                         # List files
ls -la                     # List all files with permissions and details
cd /path/to/dir            # Go to directory
cd ..                      # Go up one level
cd ~                       # Go to home directory
cd -                       # Go back to previous directory
tree                       # Visual directory tree (install: apt install tree)
```

---

## Viewing Files

```bash
cat file.txt               # Print file to terminal
less file.txt              # Scroll through file (q to quit)
head file.txt              # First 10 lines
head -n 20 file.txt        # First 20 lines
tail file.txt              # Last 10 lines
tail -f file.txt           # Follow file live (great for logs)
grep "keyword" file.txt    # Search inside file
grep -i "keyword" file.txt # Case-insensitive search
grep -n "keyword" file.txt # Show line numbers
grep -r "keyword" /dir/    # Search recursively through directory
grep -v "keyword" file.txt # Show lines NOT containing keyword
```

---

## File and Directory Operations

```bash
touch file.txt             # Create empty file
mkdir folder               # Create directory
mkdir -p a/b/c             # Create nested directories

cp file1 file2             # Copy file
cp -r folder1 folder2      # Copy entire folder
mv file1 /path/dest        # Move file
mv old.txt new.txt         # Rename file

rm file.txt                # Delete file
rm -r folder/              # Delete folder recursively
rm -rf folder/             # Force delete, no confirmation

find / -name "filename"    # Find file by name anywhere
find /home -name "*.txt"   # Find all .txt in /home
find / -mtime -1           # Files modified in last 24 hours
locate filename            # Fast indexed search
```

---

## Permissions

```bash
# Permission format: -rwxr-xr--
# Position 1:    - file | d directory | l symlink
# Positions 2-4: owner permissions (rwx)
# Positions 5-7: group permissions (r-x)
# Positions 8-10: others permissions (r--)

# Numeric values:  r=4  w=2  x=1  -=0

chmod 755 file             # rwxr-xr-x  (standard executable)
chmod 644 file             # rw-r--r--  (standard text file)
chmod 600 file             # rw-------  (private, e.g. SSH key)
chmod 700 folder           # rwx------  (private directory)
chmod +x script.sh         # Add execute for all
chmod u+x script.sh        # Add execute for owner only
chmod g-w file.txt         # Remove write from group

chown user file            # Change file owner
chown user:group file      # Change owner and group
chown -R user /folder/     # Recursive ownership change

# Find dangerous permissions
find / -perm -o+w -type f 2>/dev/null    # World-writable files
find / -perm -4000 -type f 2>/dev/null   # SUID files
```

---

## Package Management (Debian/Kali)

```bash
sudo apt update                     # Refresh package list (always first)
sudo apt upgrade -y                 # Install all updates
sudo apt install packagename        # Install package
sudo apt install -y packagename     # Install without prompt
sudo apt remove packagename         # Remove package
sudo apt purge packagename          # Remove + delete config files
sudo apt autoremove                 # Remove unused dependencies
apt search keyword                  # Search for package
apt show packagename                # Show package details

dpkg -i package.deb                 # Install .deb file directly
dpkg -l                             # List all installed packages
dpkg -l | grep packagename          # Check if specific package installed
dpkg -L packagename                 # List files installed by package
```

---

## Networking

```bash
ip a                               # Show all interfaces and IPs
ip a show eth0                     # Specific interface info
ip route                           # Show routing table
ip route show default              # Show default gateway
ifconfig                           # Alternative (older systems)

ping host -c 4                     # Send 4 pings
traceroute host                    # Show hops to destination
traceroute -n host                 # Show IPs only (faster)

netstat -tuln                      # Show listening ports
netstat -tulnp                     # Listening ports + process names
ss -tuln                           # Modern alternative to netstat
ss -tulnp                          # With process names

# DNS
nslookup domain.com                # Basic DNS lookup
dig domain.com                     # Detailed DNS query
dig domain.com MX                  # Mail records
dig domain.com TXT                 # Text records
dig -x 8.8.8.8                    # Reverse lookup
dig @8.8.8.8 domain.com           # Use specific DNS server

# Packet capture
sudo tcpdump -i eth0               # Capture on interface
sudo tcpdump -i eth0 host x.x.x.x # Filter by host
sudo tcpdump -i eth0 port 80       # Filter by port
sudo tcpdump -i eth0 -w out.pcap   # Save to file
```

---

## Users and Privileges

```bash
whoami                             # Current username
id                                 # UID, GID, group memberships
who                                # Who is logged in
last                               # Login history
w                                  # Logged-in users and activity

su username                        # Switch user
sudo command                       # Run command as root
sudo -i                            # Open root shell
sudo -l                            # List your sudo permissions

cat /etc/passwd                    # All system users
cat /etc/shadow                    # Password hashes (root only)
cat /etc/group                     # All groups

sudo useradd -m newuser            # Create user with home dir
sudo passwd newuser                # Set user password
sudo userdel -r newuser            # Delete user and home dir
```

---

## Processes

```bash
ps aux                             # All running processes
ps aux | grep processname          # Find specific process
top                                # Live process monitor
htop                               # Better live monitor (install: apt install htop)

kill PID                           # Kill process by PID
kill -9 PID                        # Force kill
killall processname                # Kill all processes by name

command &                          # Run in background
jobs                               # List background jobs
fg 1                               # Bring job 1 to foreground
nohup command &                    # Run immune to hangup signals
```

---

## Text Processing (Log Analysis)

```bash
# Sorting and counting
sort file.txt                      # Sort alphabetically
sort -n file.txt                   # Sort numerically
sort -r file.txt                   # Reverse sort
sort | uniq                        # Remove duplicates
sort | uniq -c                     # Count occurrences
sort | uniq -c | sort -rn          # Count and sort by frequency

# Cutting columns
cut -d: -f1 /etc/passwd            # First field, colon delimiter
cut -d, -f2 file.csv               # Second field, comma delimiter

# AWK
awk '{print $1}' file              # Print first column
awk -F: '{print $1}' /etc/passwd   # Colon-separated, first field
awk '{print $1,$3}' file           # Print columns 1 and 3

# SED
sed 's/old/new/g' file             # Replace all occurrences
sed -n '5,10p' file                # Print lines 5-10
sed '/pattern/d' file              # Delete lines matching pattern

# Pipes — combine commands
cat file | grep "error" | wc -l                    # Count error lines
grep "Failed" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn
# Top IPs with failed SSH attempts
```

---

## Archiving and Compression

```bash
tar -cvf archive.tar folder/       # Create tar archive
tar -xvf archive.tar               # Extract tar archive
tar -czvf archive.tar.gz folder/   # Create gzip compressed archive
tar -xzvf archive.tar.gz           # Extract gzip archive
tar -cjvf archive.tar.bz2 folder/  # Create bzip2 archive
tar -xjvf archive.tar.bz2          # Extract bzip2 archive

zip -r archive.zip folder/         # Zip a folder
unzip archive.zip                  # Unzip
```

---

## System Information

```bash
uname -a                           # Kernel version and system info
uname -r                           # Just kernel version
hostname                           # Machine hostname
uptime                             # System uptime and load
df -h                              # Disk usage (human readable)
du -sh folder/                     # Folder size
free -h                            # RAM usage (human readable)
lscpu                              # CPU information
lsblk                              # Block devices (disks)
env                                # Environment variables
echo $PATH                         # Show PATH variable
history                            # Command history
history | grep nmap                # Search command history
```

---

## Services

```bash
systemctl status servicename       # Check service status
systemctl start servicename        # Start service
systemctl stop servicename         # Stop service
systemctl restart servicename      # Restart service
systemctl enable servicename       # Start on boot
systemctl disable servicename      # Don't start on boot
systemctl list-units --type=service # List all services

service --status-all               # List all services (older syntax)
service apache2 start              # Start Apache (older syntax)
```

---

## Shortcuts and Tips

```bash
Ctrl+C          Kill current process
Ctrl+Z          Suspend current process
Ctrl+D          Exit/logout
Ctrl+L          Clear terminal (same as: clear)
Ctrl+A          Jump to beginning of line
Ctrl+E          Jump to end of line
Ctrl+R          Search command history
Tab             Autocomplete command or path
!!              Repeat last command
!nmap           Repeat last nmap command
sudo !!         Run last command with sudo

# Redirect output
command > file.txt          # Write output to file (overwrite)
command >> file.txt         # Append output to file
command 2>/dev/null         # Discard errors
command 2>&1 | tee out.txt  # Show output AND save to file

# Multiple commands
command1 && command2        # Run command2 only if command1 succeeds
command1 || command2        # Run command2 only if command1 fails
command1 ; command2         # Run both regardless
```

---

## Common Log File Locations

```bash
/var/log/syslog            # General system logs
/var/log/auth.log          # Authentication logs (SSH, sudo, logins)
/var/log/kern.log          # Kernel logs
/var/log/apache2/          # Apache web server logs
/var/log/nginx/            # Nginx web server logs
/var/log/mysql/            # MySQL logs
/var/log/ufw.log           # Firewall logs
/var/log/dpkg.log          # Package installation logs
/var/log/faillog           # Failed login attempts
~/.bash_history            # Current user's command history
```

---

## Security-Relevant One-Liners

```bash
# Find SUID binaries (privilege escalation check)
find / -perm -4000 -type f 2>/dev/null

# Find world-writable files
find / -perm -o+w -type f 2>/dev/null

# Failed SSH login attempts and source IPs
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Successful logins
grep "Accepted password" /var/log/auth.log

# Currently listening services
ss -tulnp

# Established connections
ss -tnp state established

# Check crontabs (scheduled tasks — persistence mechanism)
crontab -l
cat /etc/crontab
ls /etc/cron.*

# Check running processes for suspicious activity
ps aux --sort=-%cpu | head -20

# Check recently modified files (last 24h)
find / -mtime -1 -type f 2>/dev/null | grep -v proc
```

---

                                                        *Author: Professorshubhx |
