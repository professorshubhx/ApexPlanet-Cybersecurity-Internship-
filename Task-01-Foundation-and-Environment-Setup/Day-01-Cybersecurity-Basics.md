
# 🛡️ Day 01 — Cybersecurity Basics
### ApexPlanet Internship | Task 1: Foundation & Environment Setup
**Timeline:** Days 1–12 | **Author:** Professorshubhx

---

## 📌 What We're Covering Today

Before you touch any tool or terminal, you need to understand *why* cybersecurity exists in the first place. Today is all about building that mental foundation — the concepts that literally every security professional talks about, every single day.

---

## 1. 🔐 The CIA Triad — The Heart of Cybersecurity

If someone asks you "what is cybersecurity actually about?" — the answer is three words: **Confidentiality, Integrity, Availability**. These three things are what every attack tries to break and every defense tries to protect.

---

### 🔵 Confidentiality — "Only the right people should see it"

Confidentiality means keeping information away from people who aren't supposed to have it. Simple as that.

Think of it like this — your bank password. You know it. Your bank's server knows it (in hashed form). Nobody else should. If someone gets it without permission, *confidentiality is broken*.

**Real-world examples of confidentiality being attacked:**
- Someone steals your login credentials through phishing
- A hacker intercepts unencrypted Wi-Fi traffic (Man-in-the-Middle)
- An employee leaks customer data to a competitor

**How we protect it:**
- Encryption (so even if data is stolen, it's unreadable)
- Access controls (only authorized users get in)
- Multi-factor authentication

---

### 🟢 Integrity — "The data should be exactly what it's supposed to be"

Integrity means data hasn't been tampered with — it's accurate and trustworthy. If someone changes a file, a database record, or even a single bit of a message without you knowing, integrity is gone.

**Real-world examples:**
- An attacker modifies a bank transaction (changes ₹100 to ₹10,000)
- Malware alters system files without the user knowing
- Someone intercepts an email and changes the content before it reaches you

**How we protect it:**
- Hashing (MD5, SHA256) — creates a digital fingerprint of data; any change = different hash
- Digital signatures — prove who sent something AND that it wasn't changed
- Version control and file integrity monitoring tools

---

### 🔴 Availability — "The system should work when you need it"

Availability means systems and data are accessible to authorized users when they need them. A system that's locked down so hard that nobody can use it — that's not secure, that's broken.

**Real-world examples:**
- A DDoS attack floods a website with traffic until it crashes
- Ransomware encrypts all your files, making them inaccessible
- A power failure takes down a server with no backup

**How we protect it:**
- Redundancy and backups
- DDoS protection services
- Load balancers and failover systems
- Disaster recovery planning

---

### 🧠 Quick Memory Trick for CIA Triad:

```
C → Can others see it?         (Confidentiality)
I → Is the data untouched?     (Integrity)
A → Is the system working?     (Availability)
```

---

## 2. ⚠️ Threat Types — What Attackers Actually Do

Now let's talk about *how* attackers break the CIA Triad. These are the most common threats you'll deal with as a cybersecurity professional.

---

### 🎣 Phishing

Phishing is basically digital deception. The attacker pretends to be someone you trust — your bank, your boss, Google — to trick you into giving up sensitive info or clicking a malicious link.

It's the #1 attack vector in the world. Why? Because it's cheap, scalable, and it targets humans, not machines.

**Types:**
- **Email Phishing** — mass fake emails (e.g., "Your account is suspended, click here")
- **Spear Phishing** — targeted attack on a specific person (e.g., your CEO)
- **Smishing** — phishing via SMS
- **Vishing** — phishing via phone call

**Red flags to spot:**
- Urgent language ("Act now!", "Your account will be closed!")
- Suspicious sender email (support@paypa1.com instead of paypal.com)
- Links that don't match the display text

---

### 🦠 Malware

Malware = **Mal**icious Soft**ware**. It's an umbrella term for any software designed to harm, spy on, or take control of a system.

**Types of Malware:**

| Type | What it does |
|------|-------------|
| **Virus** | Attaches to files, spreads when files are shared |
| **Worm** | Spreads itself across networks without human interaction |
| **Trojan** | Disguised as legit software, opens a backdoor |
| **Spyware** | Silently monitors and steals your activity |
| **Adware** | Floods you with ads (annoying but sometimes dangerous) |
| **Rootkit** | Hides deep in the OS, gives attacker persistent access |
| **Keylogger** | Records every key you type (passwords, messages, everything) |

---

### 💥 DDoS — Distributed Denial of Service

Imagine thousands of people all trying to enter a single-door shop at the same time. Nobody gets in. That's DDoS.

Attackers use a **botnet** (army of infected computers) to flood a target server with so much fake traffic that it collapses and real users can't access it.

**Example:** In 2016, the Mirai botnet took down major websites like Twitter, Netflix, and Reddit by attacking Dyn DNS.

**DDoS vs DoS:**
- **DoS** = one attacker, one source
- **DDoS** = thousands of sources (botnet), much harder to stop

---

### 💉 SQL Injection

If a web app doesn't properly filter user input, an attacker can inject SQL code into a login form or search box to manipulate the database directly.

**Classic example:**
```sql
-- Normal login query:
SELECT * FROM users WHERE username='admin' AND password='1234';

-- Attacker enters: ' OR '1'='1
-- Becomes:
SELECT * FROM users WHERE username='' OR '1'='1' AND password='';
-- This returns ALL users because '1'='1' is always true!
```

This is why input validation and parameterized queries matter so much.

---

### 🔨 Brute Force

It's exactly what it sounds like — trying every possible password combination until one works.

**Types:**
- **Pure Brute Force** — tries every combination (aaa, aab, aac...)
- **Dictionary Attack** — uses a wordlist of common passwords
- **Credential Stuffing** — uses leaked username/password combos from other breaches

**Defense:** Strong passwords, account lockouts, MFA, CAPTCHA

---

### 🔒 Ransomware

Ransomware is malware that encrypts all your files, then demands payment (usually in crypto) to give you the decryption key.

**Famous examples:**
- **WannaCry (2017)** — hit 200,000+ computers in 150 countries, including hospitals
- **NotPetya** — caused $10 billion in damage worldwide
- **LockBit** — still active, targets corporations

**Defense:** Regular backups (offline/air-gapped), patch management, network segmentation

---

## 3. 🚪 Attack Vectors — How Attackers Get In

An **attack vector** is the *path or method* an attacker uses to gain unauthorized access. Knowing these helps you think from an attacker's perspective (which is literally what a SOC analyst needs to do).

---

### 🧠 Social Engineering

This isn't hacking machines — it's hacking *humans*. Social engineering exploits trust, fear, authority, and urgency to manipulate people into doing something they shouldn't.

**Techniques:**
- **Pretexting** — attacker creates a fake scenario ("I'm from IT, I need your password to fix your account")
- **Baiting** — leaving an infected USB drive in a parking lot hoping someone picks it up
- **Quid Pro Quo** — offering something in exchange for info ("Free tech support if you give us access")
- **Tailgating** — physically following someone into a restricted area

> 💡 **Key insight:** Even the most technically secure system can be compromised if you can trick one employee.

---

### 📡 Wireless Attacks

Wi-Fi is a huge attack surface, especially on public or poorly configured networks.

**Common wireless attacks:**

| Attack | How it works |
|--------|-------------|
| **Evil Twin** | Attacker sets up a fake Wi-Fi hotspot with the same name as a legit one |
| **Man-in-the-Middle (MitM)** | Attacker intercepts traffic between you and the router |
| **WPA2 Cracking** | Capturing the 4-way handshake and cracking the password offline |
| **Deauthentication Attack** | Forcing a device to disconnect so it reconnects through attacker's AP |
| **Packet Sniffing** | Capturing unencrypted traffic on a network |

**Tools used (that you'll practice later):** Aircrack-ng, Wireshark, Kismet

---

### 🏢 Insider Threats

Not all attacks come from outside. Sometimes the danger is already inside the building.

**Types of insider threats:**
- **Malicious Insider** — a disgruntled employee who intentionally leaks or destroys data
- **Negligent Insider** — an employee who accidentally causes a breach (clicking phishing link, weak password, losing a laptop)
- **Compromised Insider** — an employee whose credentials were stolen and are being used by an outsider

**Why it's hard to detect:** Insiders already have legitimate access, so their activity doesn't always look suspicious.

**Defense:** Least privilege principle, user behavior analytics (UEBA), monitoring, background checks

---

## 📊 Quick Reference Summary

```
┌─────────────────────────────────────────────────────────┐
│                    CIA TRIAD                            │
│  Confidentiality → Integrity → Availability             │
├─────────────────────────────────────────────────────────┤
│                   THREAT TYPES                          │
│  Phishing | Malware | DDoS | SQL Injection              │
│  Brute Force | Ransomware                               │
├─────────────────────────────────────────────────────────┤
│                  ATTACK VECTORS                         │
│  Social Engineering | Wireless Attacks | Insider Threats│
└─────────────────────────────────────────────────────────┘
```

---

## 🧪 Self-Check Questions

Test yourself before moving to Day 2:

1. What does each letter in CIA stand for — and give one real-world example of each being attacked?
2. What's the difference between a Virus and a Worm?
3. How does a DDoS attack differ from a regular DoS?
4. Explain SQL Injection like you're explaining it to a non-technical friend.
5. What's the difference between a malicious insider and a negligent insider?

---

## 📚 Resources Used

- NIST Cybersecurity Framework
- OWASP Top 10
- CompTIA Security+ Study Material
- Defronix Academy — Mentored by Nitesh Singh Sir

---

## 🔗 Connect With Me

- 📺 YouTube: [@Professorshubhx](https://youtube.com/@Professorshubhx)
- 💼 LinkedIn: [Professorshubhx](https://linkedin.com/in/Professorshubhx)
- 🎓 Learning at: Defronix Cybersecurity Academy

---

*Day 01 Complete ✅ | Next → Day 02: Lab Environment Setup*
