# Lab Writeup: Broken Brute-Force Protection — IP Block

> **Platform:** PortSwigger Web Security Academy  
> **Category:** Authentication  
> **Difficulty:** Practitioner  
> **Status:** ✅ Solved  
> **Date:** April 2026  

---

## Overview

This lab demonstrates a logic flaw in brute-force protection. The application blocks an IP after a set number of failed login attempts — but resets the counter whenever a **successful login** occurs. By alternating between your own valid credentials and the victim's account in the payload list, the counter is reset before the block triggers, allowing unlimited brute-force attempts.

**Objective:** Brute-force Carlos's password by bypassing the IP block protection, then log in and access his account page.

![Lab Description](step01.png)

---

## Vulnerability Description

| Attribute | Detail |
|-----------|--------|
| **Vulnerability Type** | Broken Brute-Force Protection — Logic Flaw |
| **OWASP Category** | A07:2021 – Identification and Authentication Failures |
| **Root Cause** | IP block counter resets on any successful login — even from a different account |
| **Bypass Technique** | Alternate own valid credentials with victim credentials in the attack payload |
| **Impact** | Unlimited brute-force attempts against any account |

---

## Tools Used

- **Burp Suite Intruder** – Pitchfork attack with alternating username/password payloads
- **Browser** – PortSwigger lab environment

---

## Exploitation Steps

### Step 1 — Understand the Protection Logic

The application blocks the IP after 3 consecutive failed login attempts. However, a successful login for **any** account resets this counter — this is the exploitable logic flaw.

![Lab description showing credentials](step01.png)

---

### Step 2 — Capture the Login Request

Log in with the provided credentials (`wiener:peter`) and capture the POST request in Burp Proxy. Send to **Intruder**.

![Login request captured in Burp](step02.png)

---

### Step 3 — Configure Pitchfork Attack

Set **two** payload positions — one on the username field, one on the password field. Select **Pitchfork** attack type (pairs payloads 1:1).

![Intruder Pitchfork setup](step03.png)

---

### Step 4 — Build the Username Payload List

Create a list that alternates between `wiener` (your account) and `carlos` (victim):

```
wiener
carlos
wiener
carlos
... (repeat 100+ times)
```

This ensures every other attempt is a successful `wiener` login that resets the IP counter.

![Username payload list](step04.png)

---

### Step 5 — Build the Password Payload List

Create a corresponding list that pairs `peter` (your password) with each candidate password from the wordlist:

```
peter
password1
peter
password2
peter
password3
...
```

![Password payload list](step05.png)

---

### Step 6 — Set Resource Pool to 1 Concurrent Request

Go to **Resource Pool** and set **Maximum concurrent requests** to `1`. This ensures requests are sent in order — critical for the alternating pattern to work correctly.

![Resource pool set to 1](step06.png)

---

### Step 7 — Run the Attack

Start the attack. Monitor results — look for a `302 Found` status code on a `carlos` row, which indicates a successful login.

![Attack running](step07.png)

---

### Step 8 — Identify the Correct Password

Filter results by status `302` on a Carlos row. The paired password in that row is Carlos's password.

![302 response identifying correct password](step08.png)

---

### Step 9 — Log In as Carlos

Use the discovered credentials to log in as Carlos via the browser.

![Logging in as Carlos](step09.png)

---

### Step 10 — Access Account Page

Navigate to Carlos's account page to confirm access.

![Carlos account page](step10.png)

---

### Step 11 — Lab Solved

Lab is marked as solved.

![Lab Solved](step11.png)

---

## Root Cause Analysis

```
Vulnerable Protection Logic:
  failed_attempts = 0
  
  On failed login:  failed_attempts += 1
                    if failed_attempts >= 3: block_ip()
  
  On successful login: failed_attempts = 0  ← FLAW: resets for ANY account

Exploit:
  Attempt 1: wiener:peter      → SUCCESS → counter resets to 0
  Attempt 2: carlos:password1  → FAIL    → counter = 1
  Attempt 3: wiener:peter      → SUCCESS → counter resets to 0
  Attempt 4: carlos:password2  → FAIL    → counter = 1
  ... never reaches 3 → never blocked
```

---

## Remediation

| Recommendation | Description |
|----------------|-------------|
| **Track failed attempts per username, not just per IP** | Block the target account after N failures, regardless of IP |
| **Don't reset the counter on successful logins of other accounts** | The counter for `carlos` should not reset when `wiener` logs in |
| **Implement account lockout separately from IP block** | Lock the victim account independently of who is making requests |
| **Use CAPTCHA or MFA** | Add friction that cannot be automated around |
| **Log and alert on alternating account patterns** | Flag behavior that switches between accounts rapidly |

---

## Key Takeaways

- **Brute-force protection must be tracked per target username**, not just per IP.
- **Logic flaws are often more subtle than technical bugs.** The counter reset on success seems reasonable in isolation — but it creates an exploitable bypass.
- **Pitchfork attack type in Burp Intruder** pairs two payload lists 1:1, making it ideal for synchronized username/password attacks.
- **Setting concurrent requests to 1** is critical — out-of-order requests would break the alternating pattern and trigger the IP block.

---

*Writeup produced as part of PortSwigger Web Security Academy lab practice.*
