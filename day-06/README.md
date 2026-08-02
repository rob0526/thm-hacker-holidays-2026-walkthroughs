# TryHackMe — Hacker Holidays 2026
## Day 6: [Overheard at Breakfast](https://tryhackme.com/room/hh-overheardatbreakfast-6f01793c)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** OSINT  
**Difficulty:** Easy

---

## Description

A screenshot of a private chat conversation was captured at the hotel breakfast terrace. Somewhere in the conversation, the subject reveals enough identifying information to locate an account they never meant to expose. The goal is to read carefully, extract the clues, and track down the hidden profile.

---

## Step 1 — Analyse the conversation screenshot

The screenshot shows a chat between two users: **Ponzi (Influencer)** and **Lambo**.

Reading every message carefully, two key details stand out in Lambo's messages:

> *"I used to use this free tool that let me upload my profile and link other media accounts was neat, until I wiped everything. **Started with a G** if I remember correctly."*

> *"But if anything this is my best way of communication: **lambobytelotushotel@gmail.com**"*

**Explanation:** The hint from @0xMia was *"READ what they said, not just skim it"* — this is why. The email address and the platform hint are easy to miss on a casual read. Together they are everything we need.

---

## Step 2 — Identify the platform ("starts with G")

The description matches **Gravatar** (Globally Recognised Avatar):

- ✅ Free service
- ✅ Lets you upload a profile photo
- ✅ Lets you link other social media accounts
- ✅ Starts with **G**

Gravatar is widely used as a universal profile tied to an email address. Crucially, **Gravatar profiles are public by default** and are indexed by the MD5 hash of the account's email address — even after the user "wipes" their linked accounts, the profile URL remains discoverable.

---

## Step 3 — Compute the MD5 hash of the email

Gravatar derives profile URLs from the lowercase MD5 hash of the email address.

```bash
echo -n "lambobytelotushotel@gmail.com" | md5sum
```

**Output:**

```text
<md5hash>  -
```

**Explanation:**
- `echo -n` → prints the string without a trailing newline (critical — a newline would change the hash)
- `md5sum` → computes the MD5 digest

---

## Step 4 — Visit the Gravatar profile

Navigate to:

```
https://www.gravatar.com/<md5hash>
```

The profile page for **Lambo** loads:

```text
Lambo
Lam-boh
·
Byte Lotus Hotel

Funny thing about email hashes, they follow you places you didn't expect.
Glad you found the right corner of the internet!
Here is your prize: <md5hash>
```

---

## Step 5 — Decode the Base64 prize

The prize string is Base64-encoded:

```bash
echo "<md5hash>" | base64 -d
```

**Output:**

```text
THM{[REDACTED]}
```

**Explanation:** Base64 is not encryption — it is an encoding scheme that represents binary data as printable ASCII. It is trivially reversible without any key. The `base64 -d` flag decodes the string back to its original form.

---

## Command Option Breakdown

### `echo -n`
- `-n` → do not append a newline character
- **Critical for hashing:** `echo "text"` adds `\n` at the end, which would produce a different MD5 than `echo -n "text"`

### `md5sum`
- Computes the MD5 digest of its input
- Output format: `<hash>  <filename>` (two spaces, then `-` for stdin)

### `base64 -d`
- `-d` → decode Base64 input back to original bytes

---

## Root Cause

Gravatar profiles are **permanently public and indexed by email hash**. Even after a user removes linked accounts or stops using the service, the profile URL remains live and discoverable by anyone who knows (or can guess) the associated email address. Lambo shared their email openly in a chat, not realising that it was also the key to a profile they thought they had abandoned.

### Remediation

| Issue | Fix |
| --- | --- |
| Public Gravatar profile with sensitive content | Delete the Gravatar account entirely, not just unlink accounts |
| Email shared in chat | Treat email addresses as semi-public identifiers; never embed secrets in profiles tied to them |
| Flag stored in a public profile bio | Never store sensitive data in any public-facing profile field |

---

## Result

Challenge solved by:
1. reading the conversation carefully to extract the email address and platform hint,
2. identifying Gravatar as the "free tool starting with G",
3. computing the MD5 hash of the email to construct the profile URL,
4. visiting the Gravatar profile to retrieve the Base64-encoded flag,
5. decoding Base64 to recover the flag.
