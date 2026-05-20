# Day 17 — Web Security Headers
### ApexPlanet Internship | Task 3: Web Application Security
**Timeline:** Days 25–36 | **Author:** Professorshubhx

---

## What are Security Headers

HTTP response headers are metadata that a server sends along with every web page response. Most headers control how the browser renders or caches the page. Security headers specifically tell the browser how to behave in ways that protect users from common attacks.

They are one of the easiest and most effective security controls to implement — a few lines in your web server config can prevent entire classes of vulnerabilities like XSS, clickjacking, and protocol downgrade attacks.

Many organizations skip them because they're invisible to users — but attackers check for them immediately. Tools like Burp Suite, Nikto, and securityheaders.com flag missing headers as findings in every security assessment.

---

## Checking Headers on a Live Site

### Using securityheaders.com

The fastest way to analyze any site's security headers:

```
1. Go to https://securityheaders.com/
2. Enter a URL — try: https://dvwa.co.uk or http://192.168.56.104
3. Click Scan
4. Read the report — each missing header is flagged
```

The report gives a letter grade (A+ to F) and lists:
- Headers present (green)
- Headers missing (red/orange)
- Headers with misconfigurations (yellow)

### Using curl

```bash
# View response headers only
curl -I http://192.168.56.104

# View headers with verbose output
curl -v http://192.168.56.104 2>&1 | grep -E "^[<>]"

# Check a specific header
curl -I https://google.com | grep -i "strict-transport"
curl -I https://google.com | grep -i "content-security"

# Check DVWA (with authentication)
curl -I http://192.168.56.104/dvwa/ \
  --cookie "PHPSESSID=yourid; security=low"
```

### Using Burp Suite

Intercept any response → look at the response headers in the top pane. Missing security headers are immediately visible.

### Using Nikto

```bash
# Nikto automatically checks for missing security headers
nikto -h http://192.168.56.104

# Output will include findings like:
# + The anti-clickjacking X-Frame-Options header is not present
# + The X-Content-Type-Options header is not set
```

---

## 1. Content-Security-Policy (CSP)

### What it Does

CSP tells the browser which sources are allowed to load scripts, styles, images, and other resources. Even if an attacker injects a `<script>` tag via XSS, the browser refuses to execute it if it doesn't come from an approved source.

It is the single most powerful defense against XSS.

### Header Format

```
Content-Security-Policy: directive source; directive source;
```

### Common Directives

```
default-src     Fallback for all resource types not explicitly specified
script-src      Allowed sources for JavaScript
style-src       Allowed sources for CSS
img-src         Allowed sources for images
connect-src     Allowed sources for fetch, XHR, WebSocket
font-src        Allowed sources for fonts
frame-src       Allowed sources for iframes
object-src      Allowed sources for plugins (Flash etc)
base-uri        Restricts <base> tag URLs
form-action     Restricts where forms can submit
```

### Common Source Values

```
'self'          Same origin as the document
'none'          Block everything
'unsafe-inline' Allow inline scripts/styles (weakens CSP significantly)
'unsafe-eval'   Allow eval() (weakens CSP significantly)
https:          Allow any HTTPS source
data:           Allow data: URIs
nonce-xyz       Allow specific inline script with matching nonce
https://cdn.example.com  Allow specific domain
```

### Example Policies

```apache
# Basic CSP — only allow resources from same origin
Content-Security-Policy: default-src 'self'

# Allow scripts from self and a CDN
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.jsdelivr.net

# Strict policy — block everything except self, no inline scripts
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'

# Report-Only mode — test before enforcing (logs violations, doesn't block)
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

### Testing CSP

```bash
# Check if CSP is present
curl -I https://target.com | grep -i "content-security-policy"

# Evaluate CSP strength
# Go to: https://csp-evaluator.withgoogle.com/
# Paste the policy value — it shows weaknesses
```

Common CSP weaknesses:
```
'unsafe-inline'     Allows inline scripts — defeats XSS protection
'unsafe-eval'       Allows eval() — dangerous
Wildcards (*)       Too permissive — allows any source
Missing object-src  Flash/plugin injection possible
Missing base-uri    Base tag injection possible
```

---

## 2. Strict-Transport-Security (HSTS)

### What it Does

HSTS tells the browser to only connect to this site over HTTPS — never HTTP. Even if a user types `http://` or clicks an HTTP link, the browser automatically upgrades to HTTPS without making the insecure request.

This prevents SSL stripping attacks where a MitM attacker downgrades HTTPS to HTTP to intercept traffic.

### Header Format

```
Strict-Transport-Security: max-age=seconds; includeSubDomains; preload
```

### Parameters

```
max-age             How long (seconds) browser remembers to use HTTPS
                    31536000 = 1 year (recommended minimum)

includeSubDomains   Apply HSTS to all subdomains too
                    (important — without this, subdomain.site.com could be attacked)

preload             Submit site to browser's HSTS preload list
                    Browser knows to use HTTPS even on first visit
                    Requires: max-age ≥ 31536000, includeSubDomains, HTTPS working
```

### Example

```apache
# Recommended HSTS header
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# Basic (1 year, no subdomains)
Strict-Transport-Security: max-age=31536000
```

### Adding in Apache

```apache
# In httpd.conf or .htaccess (only add if HTTPS is working)
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
```

### Important Warning

Only add HSTS after HTTPS is fully working. If you add HSTS and your HTTPS breaks, users will be locked out and can't access your site at all — there's no way to undo it until the max-age expires.

---

## 3. X-Frame-Options

### What it Does

Prevents your page from being loaded inside an `<iframe>` on another domain. This stops **clickjacking attacks** — where an attacker overlays an invisible iframe of your site on top of their page, tricking users into clicking buttons they can't see.

Classic clickjacking scenario:
```
Attacker page shows: "Click here to win a prize!" (visible button)
Behind it:           Your bank's "Confirm Transfer" button (invisible iframe)
User clicks prize → Actually clicks Confirm Transfer
```

### Header Values

```
X-Frame-Options: DENY              Never load in any iframe
X-Frame-Options: SAMEORIGIN        Only allow iframes from same origin
X-Frame-Options: ALLOW-FROM https://trusted.com   Allow specific origin (deprecated)
```

### Adding in Apache

```apache
Header always set X-Frame-Options "SAMEORIGIN"
```

### Modern Alternative — CSP frame-ancestors

```apache
# CSP frame-ancestors is the modern replacement for X-Frame-Options
Content-Security-Policy: frame-ancestors 'self'
Content-Security-Policy: frame-ancestors 'none'
Content-Security-Policy: frame-ancestors https://trusted.com
```

Use both for maximum browser compatibility.

### Testing Clickjacking

```html
<!-- Create this file and open it in browser -->
<!-- If your target loads in the iframe — it's vulnerable to clickjacking -->
<html>
<body>
<iframe src="http://192.168.56.104/dvwa/" width="800" height="600"></iframe>
</body>
</html>
```

If DVWA loads inside your iframe — no X-Frame-Options header is set — clickjacking is possible.

---

## 4. X-Content-Type-Options

### What it Does

Prevents browsers from "MIME sniffing" — guessing the content type of a response. Without this header, a browser might execute a file as JavaScript even if the server says it's a text file.

Attack scenario: An attacker uploads a file named `image.jpg` that actually contains JavaScript. Without this header, some browsers might execute it.

### Header

```
X-Content-Type-Options: nosniff
```

Always set this to `nosniff`. There is no reason not to.

### Adding in Apache

```apache
Header always set X-Content-Type-Options "nosniff"
```

---

## 5. Referrer-Policy

### What it Does

Controls how much information is included in the `Referer` header when a user navigates from your page to another. By default, the full URL is sent — which can leak sensitive information like session tokens in URLs, internal paths, or user data.

```
User is on: https://yourapp.com/dashboard?user_id=12345&token=secret
User clicks a link to: https://external-site.com
Referer header sent: https://yourapp.com/dashboard?user_id=12345&token=secret
```

That external site now has your user's token.

### Header Values

```
no-referrer                 Never send Referer header
no-referrer-when-downgrade  Send full URL for HTTPS→HTTPS, nothing for HTTPS→HTTP
origin                      Send only the origin (https://yourapp.com), not the path
origin-when-cross-origin    Full URL same-origin, only origin cross-origin
same-origin                 Only send Referer for same-origin requests
strict-origin               Send only origin, never for HTTPS→HTTP
strict-origin-when-cross-origin   Best balance — modern browser default
unsafe-url                  Always send full URL (insecure)
```

### Adding in Apache

```apache
# Recommended
Header always set Referrer-Policy "strict-origin-when-cross-origin"

# Most restrictive
Header always set Referrer-Policy "no-referrer"
```

---

## 6. Permissions-Policy (formerly Feature-Policy)

### What it Does

Controls which browser features and APIs the page is allowed to use — camera, microphone, geolocation, fullscreen, payment, etc. Restricting these prevents malicious scripts from accessing hardware even if XSS runs.

### Header Format

```
Permissions-Policy: feature=(allowlist)
```

### Example

```apache
# Disable camera, microphone, geolocation for this page
Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"

# Allow geolocation only for same origin
Header always set Permissions-Policy "geolocation=(self)"

# Strict — disable everything
Header always set Permissions-Policy "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()"
```

---

## 7. Adding Security Headers in Apache

### Method 1 — httpd.conf (Global)

```bash
# Edit Apache config
sudo nano /etc/apache2/apache2.conf

# Or the default site config
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Add inside `<VirtualHost>` or globally:

```apache
# Enable mod_headers (required)
LoadModule headers_module modules/mod_headers.so

<VirtualHost *:80>
    # Security Headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self'; object-src 'none';"
    Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"

    # Only add HSTS if HTTPS is configured
    # Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
</VirtualHost>
```

### Method 2 — .htaccess (Per Directory)

```bash
sudo nano /var/www/html/.htaccess
```

```apache
<IfModule mod_headers.c>
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Content-Security-Policy "default-src 'self';"
</IfModule>
```

### Enable mod_headers and Restart Apache

```bash
# Enable mod_headers
sudo a2enmod headers

# Test config is valid
sudo apache2ctl configtest
# Should output: Syntax OK

# Restart Apache
sudo systemctl restart apache2
```

### Verify Headers are Working

```bash
# Check headers on your Apache server
curl -I http://192.168.56.104

# Should now show:
# X-Frame-Options: SAMEORIGIN
# X-Content-Type-Options: nosniff
# Referrer-Policy: strict-origin-when-cross-origin
# Content-Security-Policy: default-src 'self';
```

---

## 8. Analyzing DVWA with securityheaders.com

Since DVWA is on a local network, use curl to analyze its headers locally:

```bash
# Check what headers DVWA currently sends
curl -I http://192.168.56.104/dvwa/

# Output (before adding security headers):
HTTP/1.1 200 OK
Date: Sun, 10 May 2026 11:00:00 GMT
Server: Apache/2.2.8 (Ubuntu) DAV/2
X-Powered-By: PHP/5.2.4-2ubuntu5.10
Content-Type: text/html; charset=utf-8
```

Notice what's missing — every security header. Also notice what's exposed:
- Server version: Apache 2.2.8
- PHP version: 5.2.4

These tell an attacker exactly what vulnerabilities to look for. Add these headers to hide server info too:

```apache
# Hide Apache version
ServerTokens Prod
ServerSignature Off

# Hide PHP version
Header always unset X-Powered-By
```

### For Public Sites — Use securityheaders.com

```
1. Go to https://securityheaders.com
2. Test a known site — try: https://owasp.org
3. Note which headers they have and which are missing
4. Try: https://google.com — compare their grade
5. Try: https://facebook.com — observe their CSP policy
```

This gives you real-world examples of how major sites implement security headers.

---

## 9. Complete Security Headers Template

This is a production-ready security headers configuration for Apache:

```apache
<VirtualHost *:443>
    ServerName yoursite.com

    # Prevent clickjacking
    Header always set X-Frame-Options "SAMEORIGIN"

    # Prevent MIME sniffing
    Header always set X-Content-Type-Options "nosniff"

    # Control referrer information
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # Content Security Policy
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self';"

    # HSTS — only for HTTPS sites
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

    # Permissions Policy
    Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"

    # Hide server information
    ServerTokens Prod
    ServerSignature Off
    Header always unset X-Powered-By

    # Remove server header completely
    Header always unset Server
</VirtualHost>
```

After adding these, running securityheaders.com should give an A or A+ grade.

---

## Practice Tasks

1. Run `curl -I http://192.168.56.104/dvwa/` — list every header present and every security header missing
2. Open `https://securityheaders.com` and scan `https://owasp.org` — note the grade and missing headers
3. Compare headers of `https://google.com` vs `https://facebook.com` — which has stricter CSP?
4. Enable `mod_headers` on Kali's Apache: `sudo a2enmod headers`
5. Add `X-Frame-Options: SAMEORIGIN` to Apache config — verify with curl after restart
6. Create a clickjacking test page with an iframe pointing to DVWA — confirm it loads (no X-Frame-Options)
7. Add `X-Frame-Options: DENY` to DVWA's Apache config — reload and confirm the iframe is now blocked
8. Add the full CSP header `default-src 'self'` — test if DVWA still loads properly (it may break some inline scripts)
9. Go to `https://csp-evaluator.withgoogle.com/` — paste `default-src 'self'; script-src 'self'` — read the evaluation
10. Run Nikto against DVWA before and after adding headers: `nikto -h http://192.168.56.104` — compare findings

---

## Summary Table — All Security Headers

| Header | Protects Against | Recommended Value |
|--------|-----------------|-------------------|
| Content-Security-Policy | XSS, injection | `default-src 'self'; object-src 'none'` |
| Strict-Transport-Security | SSL stripping, MitM | `max-age=31536000; includeSubDomains` |
| X-Frame-Options | Clickjacking | `SAMEORIGIN` |
| X-Content-Type-Options | MIME sniffing | `nosniff` |
| Referrer-Policy | Info leakage | `strict-origin-when-cross-origin` |
| Permissions-Policy | Feature abuse | `camera=(), microphone=(), geolocation=()` |

---

## Quick Reference

```
CHECK HEADERS                   ADD IN APACHE
curl -I http://target           sudo nano /etc/apache2/apache2.conf
nikto -h http://target          sudo a2enmod headers
securityheaders.com             sudo apache2ctl configtest
Burp Suite → Response           sudo systemctl restart apache2

CSP SOURCES                     VERIFY AFTER ADDING
'self'    Same origin           curl -I http://192.168.56.104
'none'    Block all             grep "X-Frame" response
'unsafe-inline'  Inline JS      grep "Content-Security" response
https://cdn.x.com  Specific URL
nonce-xyz  Specific inline block
```

---

## Resources

- securityheaders.com: https://securityheaders.com/
- CSP Evaluator: https://csp-evaluator.withgoogle.com/
- OWASP Secure Headers: https://owasp.org/www-project-secure-headers/
- MDN Security Headers Reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers
- HSTS Preload List: https://hstspreload.org/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- **LinkedIn:** [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)
- **TryHackMe:** [professorshubhx](https://tryhackme.com/p/professorshubhx)

---

*Day 17 Complete | Task 3 topics covered. Next: Security Testing Report + Task 3 README*
