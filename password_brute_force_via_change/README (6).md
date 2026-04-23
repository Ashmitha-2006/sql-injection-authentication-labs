# Lab: Password Brute-Force via Password Change

**Difficulty:** `PRACTITIONER`  
**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication — Brute-Force via Business Logic Flaw

---

## Objective

This lab's password change functionality makes it vulnerable to brute-force attacks. To solve the lab, use the list of candidate passwords to brute-force Carlos's account and access his "My account" page.

| Credential | Value |
|---|---|
| Your credentials | `wiener:peter` |
| Victim's username | `carlos` |
| Passwords | Candidate password list (from lab) |

---

## Vulnerability Explanation

The password change form accepts a `username` parameter as hidden input. The form's **error messages differ depending on whether the current password is correct or not**, even when the new passwords don't match:

| Scenario | Server Response |
|---|---|
| Wrong current password + any new passwords | `Current password is incorrect` |
| Correct current password + mismatched new passwords | `New passwords do not match` |
| Correct current password + matching new passwords (for another user) | Account locked out |

By setting the two new password fields to **different values** and iterating through candidate passwords in the `current-password` field, we can use the **"New passwords do not match"** message as a reliable oracle — it fires only when the current password is correct. This bypasses any lockout because a mismatched new password change never fully completes.

---

## Tools Required

- **Burp Suite** (Community Edition is sufficient)
- Candidate password list (provided by the lab)

---

## Step-by-Step Solution

### Step 1 — Understand the Lab & Open It

Read the lab description. The lab is marked **PRACTITIONER** difficulty and is **Not solved** at the start.

![Step 1 — Lab description and objective](step01.png)

---

### Step 2 — Log In With Your Own Account

Log in using your credentials: `wiener` / `peter`. We need to access the password change functionality as an authenticated user first.

![Step 2 — Logging in as wiener with own credentials](step02.png)

---

### Step 3 — Experiment With the Password Change Form: "New passwords do not match"

On the **My Account** page, use the password change form. Enter your **correct** current password (`peter`) and two **different** values for the new password fields (e.g., `abc` and `xyz`). Submit and observe the error message.

![Step 3 — Account page showing "New passwords do not match" with correct current password](step03.png)

> When the current password is **correct** but the two new password entries don't match, the server returns: **"New passwords do not match"**. This is our oracle signal.

---

### Step 4 — Test With Wrong Current Password

Now try submitting the form with an **incorrect** current password and two different new passwords. The error message changes.

![Step 4 — Account page showing "Current password is incorrect"](step04.png)

> When the current password is **wrong**, the server returns: **"Current password is incorrect"** — regardless of what the new passwords are. This distinguishes wrong-password attempts from right-password attempts. We will use this difference to brute-force Carlos's password.

---

### Step 5 — Capture the POST /my-account/change-password Request in Burp

With Burp running, submit a valid password change request (correct current password + two mismatched new passwords). In **Proxy → HTTP history**, locate the `POST /my-account/change-password` request and send it to **Intruder**.

![Step 5 — Burp Proxy HTTP history showing POST /my-account/change-password request](step05.png)

> The HTTP history shows multiple `POST /my-account/change-password` requests. Select the one with your valid session cookie (the highlighted row) and send it to Intruder.

---

### Step 6 — Configure Intruder: Sniper Attack + Grep Match Rule

In **Burp Intruder**, change the `username` parameter value to `carlos`. Place a payload marker on the `current-password` parameter. Set the two new password fields to **different values** so the account is never actually changed:

```
username=carlos&current-password=§incorrect-password§&new-password-1=abc&new-password-2=xyz
```

Then open the **Settings** panel and add a **Grep - Match** rule to flag responses containing the string: `New passwords do not match`

![Step 6 — Intruder configured with carlos username, payload on current-password, and Grep-Match rule](step06.png)

> The Grep-Match rule is essential — it automatically flags which response contains the oracle string, saving you from scanning through hundreds of results manually. The attack type is **Sniper** since we only have one payload position.

---

### Step 7 — Load the Password List as the Payload Set

In the **Payloads** panel, load the candidate password list provided by the lab as the payload set. The payload count should reflect the number of passwords in the list.

![Step 7 — Payloads panel with candidate password list loaded](step07.png)

> The password list includes common passwords such as `123456`, `password`, `12345678`, `qwerty`, etc. Each will be tested against Carlos's `current-password` field.

---

### Step 8 — Review the Final Intruder Configuration

Confirm the final request in the Positions panel. The `username` is hardcoded to `carlos`, the `current-password` field has the payload marker, and the new passwords are mismatched (`abc` / `xyz`):

```
username=carlos&current-password=§incorrect-password§&new-password-1=abc&new-password-2=xyz
```

Start the attack.

![Step 8 — Final Intruder request with carlos username and payload marker on current-password](step08.png)

---

### Step 9 — Identify the Flagged Response

When the attack finishes, scan the results for the row with a **`1`** in the **"New passwords do not match"** column — this is the only request where the current password was correct. Note the payload (password) for that row.

![Step 9 — Attack results: request 43 (batman) flagged with "New passwords do not match"](step09.png)

> Row 43 shows the payload `batman` returned a flag in the "New passwords do not match" column — with a slightly different response length (`4114` vs `4117` for all other rows). This confirms `batman` is Carlos's current password.

---

### Step 10 — Log In as Carlos With the Discovered Password

In the browser, log out of your `wiener` account. Navigate to the login page and enter `carlos` as the username and the discovered password.

![Step 10 — Logging in as Carlos with the discovered password](step10.png)

---

### Step 11 — Lab Solved: Carlos's Account Page

The application logs you in as Carlos and the **"Congratulations, you solved the lab!"** banner appears. Carlos's **My Account** page is now accessible.

![Step 11 — Lab solved, Carlos's My Account page](step11.png)

---

## Key Takeaway

The vulnerability exists because:

1. The password change endpoint **accepts a `username` parameter from the client** — it should use the server-side session to determine the user, not user-supplied input.
2. The **differential error messages** create an oracle that reveals whether a guess was correct without any account lockout triggering, since the password change never completes (mismatched new passwords).

This is a **business logic flaw** combined with **information disclosure through error messages**.

---

## Remediation

- The password change endpoint should **derive the username from the authenticated session**, never from a client-supplied parameter.
- Use a **generic error message** for all failure cases (e.g., "Password change failed") to prevent oracle-based enumeration.
- Implement **rate limiting** on the password change endpoint, separate from login rate limiting.
- Require re-authentication (e.g., a re-login prompt) before allowing password changes for sensitive accounts.

---

*PortSwigger Web Security Academy — Authentication Labs*
