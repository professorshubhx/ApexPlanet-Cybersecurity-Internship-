# Day 05 — Cryptography Basics
### ApexPlanet Internship | Task 1: Foundation & Environment Setup
**Timeline:** Days 1–12 | **Author:** Professorshubhx

---

## Why Cryptography Matters in Cybersecurity

Cryptography is what makes secure communication possible. Every time you log into a website, send an email, or make an online payment — cryptography is working in the background protecting that data.

As a security professional, you need to understand cryptography not just to defend systems but to understand how attacks work when crypto is implemented wrong. Weak algorithms, expired certificates, poor key management — these are real vulnerabilities that show up in penetration tests and SOC investigations constantly.

Today covers symmetric and asymmetric encryption, hashing, digital certificates, SSL/TLS, and hands-on practice with OpenSSL.

---

## 1. Symmetric Encryption

Symmetric encryption uses the same key to both encrypt and decrypt data. Think of it like a padlock where the same physical key locks and unlocks it.

```
Plaintext ──► [Encrypt with Key A] ──► Ciphertext ──► [Decrypt with Key A] ──► Plaintext
```

### How it Works

You have a message: `"Transfer $500 to account 12345"`
You apply an encryption algorithm with a secret key.
The output is ciphertext: `"x7Kp9mN2qR8vL3jT..."` — unreadable without the key.
The receiver, who has the same key, decrypts it back to the original message.

### Common Symmetric Algorithms

**AES — Advanced Encryption Standard**

AES is the gold standard for symmetric encryption. It's what your Wi-Fi (WPA2/WPA3), HTTPS, disk encryption (BitLocker, VeraCrypt), and most modern secure systems use.

```
Key sizes:
AES-128   128-bit key   Fast, still very secure
AES-256   256-bit key   Slower, used for highly sensitive data (government, military)

Modes of operation (how AES handles large data):
ECB  — Electronic Codebook       Insecure, identical blocks produce identical ciphertext
CBC  — Cipher Block Chaining     Better, each block depends on the previous
CTR  — Counter Mode              Turns AES into a stream cipher, parallelizable
GCM  — Galois/Counter Mode       Provides both encryption AND authentication (most common today)
```

**DES — Data Encryption Standard**

56-bit key. Developed in the 1970s, considered broken since 1999. A 56-bit key can be brute-forced in hours with modern hardware. You'll still see it mentioned in legacy systems and exam questions, but never use it for anything real.

**3DES — Triple DES**

Applies DES three times with different keys. More secure than DES but significantly slower than AES. Being phased out — NIST deprecated it in 2017. You'll encounter it in older banking systems and VPN configurations.

**RC4**

A stream cipher that was widely used in WEP (Wi-Fi encryption) and early SSL/TLS. Known to have serious weaknesses. Completely deprecated. If you see RC4 in a Wireshark capture, that's a finding.

### The Key Distribution Problem

Symmetric encryption has one fundamental problem — how do you securely share the key with the other party in the first place? If you send the key over an insecure channel and someone intercepts it, all your encrypted communications are compromised.

This is exactly the problem that asymmetric encryption solves.

---

## 2. Asymmetric Encryption

Asymmetric encryption uses a mathematically linked key pair — a public key and a private key. What one key encrypts, only the other can decrypt.

```
Public Key  — Share with everyone, post it publicly, no problem
Private Key — Never leaves your possession, never shared
```

### How it Works

**For confidential communication:**

```
Sender                                    Receiver
  |                                           |
  |  Gets Receiver's PUBLIC key               |
  |  Encrypts message with PUBLIC key         |
  |──────── Encrypted message ───────────────►|
  |                                           |  Decrypts with PRIVATE key
  |                                           |  (only they can do this)
```

Only the receiver's private key can decrypt what was encrypted with their public key. Even the sender can't decrypt what they just sent.

**For digital signatures (proving identity):**

```
Sender                                    Receiver
  |                                           |
  |  Signs message with PRIVATE key           |
  |──────── Message + Signature ─────────────►|
  |                                           |  Verifies with Sender's PUBLIC key
  |                                           |  (proves it came from Sender)
```

If the signature verifies with the public key, the message definitely came from the private key holder and was not tampered with.

### Common Asymmetric Algorithms

**RSA — Rivest–Shamir–Adleman**

The most widely used asymmetric algorithm. Security relies on the mathematical difficulty of factoring very large numbers (the product of two large primes).

```
Key sizes:
RSA-1024   Considered weak, don't use
RSA-2048   Current minimum standard
RSA-4096   For highly sensitive applications
```

RSA is used in HTTPS certificates, SSH key pairs, PGP email encryption, and digital signatures.

**ECC — Elliptic Curve Cryptography**

Achieves equivalent security to RSA with much smaller key sizes, making it faster and more efficient — especially important for mobile devices and IoT.

```
ECC-256 provides roughly equivalent security to RSA-3072
```

Used in modern TLS, cryptocurrency (Bitcoin uses secp256k1), and modern SSH keys.

**Diffie-Hellman Key Exchange**

Not used for encryption directly — it's a key exchange protocol. It allows two parties to establish a shared secret over an insecure channel without ever transmitting the secret itself.

This is the mechanism used in TLS to establish session keys. The "perfect forward secrecy" (PFS) you hear about with ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) means each session gets fresh keys — so capturing traffic today won't help decrypt it even if the server's private key is stolen later.

### Symmetric vs Asymmetric in Practice

In the real world, they're used together. Asymmetric encryption is computationally expensive — you wouldn't use RSA to encrypt an entire 4GB file. Instead:

```
1. Use asymmetric encryption to securely exchange a symmetric key
2. Use that symmetric key to encrypt the actual data

This is exactly what TLS does during the handshake.
```

---

## 3. Hashing

Hashing is completely different from encryption. Hashing is a one-way function — you put data in, you get a fixed-length output (the hash or digest) out. There is no key, and there is no way to reverse it.

```
Input: "password123"  ──► [SHA-256] ──► ef92b778bafe771207e4... (64 hex chars)
Input: "password124"  ──► [SHA-256] ──► 74b1e6... (completely different hash)
```

Even a single character change produces a completely different hash — this is called the **avalanche effect**.

### Properties of a Good Hash Function

- **Deterministic** — same input always produces same output
- **Fast to compute** — should be quick to hash any input
- **Pre-image resistant** — you can't reverse the hash to find the input
- **Collision resistant** — extremely unlikely that two different inputs produce the same hash
- **Avalanche effect** — tiny input change = completely different output

### Common Hash Algorithms

**MD5 — Message Digest 5**

Produces a 128-bit (32 hex character) hash. Widely used historically but now considered cryptographically broken — collisions can be generated deliberately. Never use MD5 for security purposes.

```bash
echo -n "hello" | md5sum
# Output: 5d41402abc4b2a76b9719d911017c592
```

You'll still see MD5 used for file integrity checking (non-security purposes) and in older password databases.

**SHA-1 — Secure Hash Algorithm 1**

160-bit hash. Also broken — Google demonstrated a practical collision in 2017 (the SHAttered attack). Deprecated in certificates, git is moving away from it. Don't use for security.

```bash
echo -n "hello" | sha1sum
# Output: aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d
```

**SHA-256 / SHA-512 — SHA-2 Family**

Current standard. SHA-256 produces a 256-bit hash, SHA-512 produces a 512-bit hash. No known practical attacks. This is what you should use.

```bash
echo -n "hello" | sha256sum
# Output: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

echo -n "hello" | sha512sum
```

**SHA-3**

The latest SHA standard, using a completely different internal design (Keccak sponge construction) from SHA-2. Not yet widely deployed but important to know.

**bcrypt / scrypt / Argon2**

These are specifically designed for hashing passwords. Unlike SHA-256 which is designed to be fast, these are intentionally slow and computationally expensive. This makes brute-force attacks much harder.

```
SHA-256: Can compute billions of hashes per second on a GPU
bcrypt:  Designed to compute only thousands per second
```

If a website stores passwords using SHA-256, an attacker with a leaked database can crack millions of passwords in hours. With bcrypt or Argon2, it's significantly harder. The work factor can also be increased as hardware gets faster.

### What Hashing is Used For

```
Password storage        Store hash, not plaintext. Compare hashes at login.
File integrity          Hash a file — any change produces a different hash
Digital signatures      Hash is signed, not the entire document (more efficient)
Checksums               Verify downloads aren't corrupted or tampered with
Blockchain              Each block contains hash of previous block (chain integrity)
Forensics               Hash evidence files to prove they haven't been modified
```

### Hash Cracking

Since you can't reverse a hash, attackers use:

- **Dictionary attack** — hash every word in a wordlist, compare to target hash
- **Rainbow tables** — pre-computed tables of hash-to-plaintext mappings
- **Brute force** — hash every possible combination

**Salting** defeats rainbow tables. A salt is a random value added to the password before hashing:

```
Password: "hunter2"
Salt:     "x7Kp9m" (random, stored alongside hash)
Hash:     SHA-256("hunter2x7Kp9m") = ...unique hash...
```

Even if two users have the same password, they get different hashes. Rainbow tables become useless because they'd need to be recomputed for every possible salt.

---

## 4. Digital Certificates and PKI

A digital certificate is a document that binds a public key to an identity (a person, organization, or domain). It's what makes HTTPS trustworthy — not just encrypted, but encrypted to the right server.

### The Problem Certificates Solve

Asymmetric encryption is great, but how do you know the public key you received actually belongs to google.com and not an attacker sitting between you and Google? You need a trusted third party to vouch for it.

This is what a Certificate Authority (CA) does.

### Certificate Authorities (CAs)

A CA is a trusted organization that:
1. Verifies the identity of the certificate requester
2. Signs their certificate with the CA's private key
3. Allows anyone to verify the certificate using the CA's public key

```
Google submits their public key + proof of domain ownership to DigiCert (a CA)
DigiCert verifies it, then signs the certificate
Your browser ships with DigiCert's public key pre-installed (trusted root)
When you visit google.com, your browser verifies DigiCert's signature
If valid → you're actually talking to google.com
```

### PKI — Public Key Infrastructure

PKI is the entire system of CAs, certificates, and trust relationships that makes this work at internet scale.

```
Root CA (highest trust, self-signed)
    |
    └── Intermediate CA (signed by Root CA)
            |
            └── End-Entity Certificate (your website's cert, signed by Intermediate)
```

Root CA certificates are embedded in your OS and browser. If a Root CA is compromised, millions of certificates become untrustworthy — this is why Root CAs are kept offline in highly secure facilities.

### What's Inside a Certificate (X.509 Format)

```
Subject:          CN=google.com, O=Google LLC, C=US
Issuer:           CN=GTS CA 1C3, O=Google Trust Services
Valid From:       2024-01-01
Valid To:         2024-03-31
Public Key:       [RSA or ECC public key]
Serial Number:    [unique identifier]
Signature:        [CA's digital signature over all the above]
SANs:             *.google.com, google.com, www.google.com
```

### Certificate Types

```
DV — Domain Validated    Basic. Just proves you control the domain. Issued in minutes.
OV — Organization Valid  Also validates the organization. Requires documentation.
EV — Extended Validation Strictest. Shows org name in browser (green bar, now rare).
Wildcard                 *.example.com — covers all subdomains
SAN                      Multiple domains on one cert (Subject Alternative Names)
```

### Certificate Validation Errors — What They Mean

```
Certificate expired           Cert's validity period has passed
Certificate not trusted       Signed by a CA not in your trusted root store
Hostname mismatch             Cert is for example.com but you're visiting other.com
Self-signed certificate       No CA signed it — could be legitimate (internal) or MITM
Certificate revoked           CA has revoked it (CRL or OCSP check)
```

In security assessments, self-signed certs and expired certs are common findings. In phishing investigations, a valid HTTPS certificate doesn't mean a site is legitimate — attackers get free DV certificates all the time.

---

## 5. SSL/TLS

SSL (Secure Sockets Layer) is the predecessor to TLS (Transport Layer Security). SSL is deprecated and broken — TLS is what's actually used today, but the name "SSL" has stuck colloquially.

```
SSL 2.0  — 1995, broken, never use
SSL 3.0  — 1996, broken (POODLE attack), never use
TLS 1.0  — 1999, deprecated (BEAST attack)
TLS 1.1  — 2006, deprecated
TLS 1.2  — 2008, still widely used, secure with proper configuration
TLS 1.3  — 2018, current standard, faster and more secure
```

### The TLS Handshake (TLS 1.2)

```
Client                                        Server
  |                                               |
  |── ClientHello ─────────────────────────────►|
  |   (TLS version, cipher suites, random bytes) |
  |                                               |
  |◄── ServerHello ────────────────────────────|
  |    (chosen cipher suite, server random)      |
  |                                               |
  |◄── Certificate ────────────────────────────|
  |    (server's public key + CA signature)      |
  |                                               |
  |◄── ServerHelloDone ────────────────────────|
  |                                               |
  |  [Client verifies certificate]               |
  |                                               |
  |── ClientKeyExchange ──────────────────────►|
  |   (pre-master secret, encrypted w/ server    |
  |    public key)                               |
  |                                               |
  |  [Both sides derive session keys from        |
  |   pre-master secret + random values]         |
  |                                               |
  |── ChangeCipherSpec ───────────────────────►|
  |── Finished (encrypted) ──────────────────►|
  |◄── ChangeCipherSpec ────────────────────────|
  |◄── Finished (encrypted) ────────────────────|
  |                                               |
  |════════ Encrypted Application Data ══════════|
```

TLS 1.3 simplifies this significantly — the handshake is faster (1-RTT instead of 2-RTT) and removes support for weak cipher suites entirely.

### Cipher Suites

A cipher suite is a combination of algorithms used for the TLS connection:

```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384

TLS          — Protocol
ECDHE        — Key exchange algorithm (Elliptic Curve Diffie-Hellman Ephemeral)
RSA          — Authentication algorithm (certificate signature)
AES_256_GCM  — Symmetric encryption algorithm and mode
SHA384       — Hash algorithm for message authentication (HMAC)
```

Weak cipher suites (those using RC4, DES, MD5, export-grade crypto) are a common finding in TLS configuration audits.

### Checking TLS Configuration

```bash
# Check what TLS versions and ciphers a server supports
openssl s_client -connect google.com:443

# Check certificate details
openssl s_client -connect google.com:443 | openssl x509 -text -noout

# Test specific TLS version
openssl s_client -connect google.com:443 -tls1_2
openssl s_client -connect google.com:443 -tls1_3

# Using nmap to check TLS
nmap --script ssl-enum-ciphers -p 443 google.com
```

---

## 6. Hands-On with OpenSSL

OpenSSL is a command-line toolkit for working with cryptography. It's pre-installed on Kali Linux. Practice all of these on your Kali VM.

### Generating Keys and Encrypting Files

```bash
# Generate a random 256-bit AES key (hex encoded)
openssl rand -hex 32

# Encrypt a file with AES-256-CBC
echo "This is my secret message" > secret.txt
openssl enc -aes-256-cbc -salt -in secret.txt -out secret.enc -k mypassword

# Decrypt it
openssl enc -aes-256-cbc -d -in secret.enc -out decrypted.txt -k mypassword
cat decrypted.txt

# Verify they match
sha256sum secret.txt decrypted.txt
```

### Hashing with OpenSSL

```bash
# Hash a string
echo -n "hello world" | openssl dgst -md5
echo -n "hello world" | openssl dgst -sha1
echo -n "hello world" | openssl dgst -sha256
echo -n "hello world" | openssl dgst -sha512

# Hash a file
openssl dgst -sha256 secret.txt

# Verify a download (compare hash to what the website says it should be)
openssl dgst -sha256 downloaded-file.iso
```

### Generating RSA Key Pairs

```bash
# Generate a 2048-bit RSA private key
openssl genrsa -out private.key 2048

# View the private key
cat private.key

# Extract the public key from the private key
openssl rsa -in private.key -pubout -out public.key

# View the public key
cat public.key

# See the key components (modulus, exponent, etc.)
openssl rsa -in private.key -text -noout
```

### Encrypting and Decrypting with RSA

```bash
# Create a test message
echo "Secret message for RSA" > message.txt

# Encrypt with the public key
openssl rsautl -encrypt -inkey public.key -pubin -in message.txt -out message.enc

# Decrypt with the private key
openssl rsautl -decrypt -inkey private.key -in message.enc -out message_dec.txt
cat message_dec.txt
```

### Creating a Self-Signed Certificate

```bash
# Generate a private key and self-signed certificate in one command
openssl req -x509 -newkey rsa:2048 -keyout mykey.pem -out mycert.pem -days 365 -nodes

# You'll be prompted for:
# Country Name: IN
# State: Uttar Pradesh
# City: Varanasi
# Organization: Defronix Academy
# Common Name: localhost (or your domain)

# View the certificate
openssl x509 -in mycert.pem -text -noout

# Check certificate dates
openssl x509 -in mycert.pem -noout -dates

# Check who issued it
openssl x509 -in mycert.pem -noout -issuer -subject
```

### Signing and Verifying (Digital Signatures)

```bash
# Create a document to sign
echo "I authorize this transaction" > document.txt

# Sign it with your private key (creates a signature file)
openssl dgst -sha256 -sign private.key -out document.sig document.txt

# Verify the signature using the public key
openssl dgst -sha256 -verify public.key -signature document.sig document.txt
# Output: Verified OK

# Try tampering with the document
echo "I authorize a DIFFERENT transaction" > document.txt
openssl dgst -sha256 -verify public.key -signature document.sig document.txt
# Output: Verification Failure
```

This demonstrates exactly how digital signatures work — the signature is tied to the exact content. Any change breaks verification.

---

## How It All Connects

```
HTTPS in one flow:

1. Your browser gets the server's certificate
   └── Certificate contains server's PUBLIC KEY
   └── Certificate is signed by a trusted CA (verified using CA's public key)

2. TLS handshake uses ASYMMETRIC crypto (RSA/ECDHE)
   └── To securely exchange a session key

3. All actual data is encrypted with SYMMETRIC crypto (AES-256-GCM)
   └── Because symmetric is much faster for bulk data

4. Each message is authenticated with HASHING (SHA-384 HMAC)
   └── Integrity check — detects any tampering in transit

5. The server's identity is verified via DIGITAL CERTIFICATE
   └── Signed by a CA you already trust
```

Encryption without authentication = vulnerable to tampering.
Authentication without encryption = vulnerable to eavesdropping.
TLS provides both, which is why it's the foundation of secure internet communication.

---

## Practice Tasks

Complete these on your Kali VM:

1. Encrypt a text file using `openssl enc -aes-256-cbc` then decrypt it and confirm contents match
2. Hash the same file with MD5, SHA-1, and SHA-256 — compare the output lengths
3. Change one character in the file and re-hash — observe the avalanche effect
4. Generate an RSA-2048 key pair, view both keys, encrypt a message with the public key and decrypt with the private key
5. Create a self-signed certificate, view it with `openssl x509 -text -noout`, and identify the subject, issuer, and validity dates
6. Sign a document with your private key, verify it, then modify the document and verify again — observe the failure
7. Run `openssl s_client -connect google.com:443` and identify: TLS version used, cipher suite, and certificate expiry date
8. Look up what cipher suite your Kali VM's Apache uses (if running): `nmap --script ssl-enum-ciphers -p 443 localhost`

---

## Quick Reference

```
SYMMETRIC                   ASYMMETRIC                  HASHING
Same key encrypt/decrypt    Public/private key pair     One-way, no decryption
AES-256 (current standard)  RSA-2048+ (current)         SHA-256 (current)
Fast, for bulk data         Slow, for key exchange      Used for integrity
DES/3DES (legacy/broken)    ECC (modern, efficient)     MD5/SHA-1 (broken)

CERTIFICATES                TLS VERSIONS                OPENSSL COMMANDS
X.509 format                TLS 1.0/1.1 — deprecated   openssl enc (encrypt)
DV/OV/EV types              TLS 1.2 — acceptable        openssl dgst (hash)
Signed by CA                TLS 1.3 — current           openssl genrsa (keygen)
CN, SAN, validity dates     SSL 2/3 — completely broken openssl s_client (connect)
```

---

## Resources

- Cryptography explained visually: https://www.crypto101.io/
- OpenSSL docs: https://www.openssl.org/docs/
- TLS configuration best practices: https://ssl-config.mozilla.org/
- SSL Labs (test any website's TLS config): https://www.ssllabs.com/ssltest/
- Defronix Academy | Mentor: Nitesh Singh Sir

---

## Connect

- Tryhackme: [professorshubhx](https://github.com/p/professorshubhx)
- LinkedIn: [Shubham Chaurasiya](https://www.linkedin.com/in/shubham-chaurasiya-60932a359/)

---

*Day 05 Complete | Next: Day 06 — Tool Familiarization (Wireshark, Nmap, Burp Suite, Netcat)*
