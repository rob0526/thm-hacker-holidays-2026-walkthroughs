# TryHackMe — Hacker Holidays 2026
## Day 4: [Packed Light](https://tryhackme.com/room/hh-packedlight-02e5330c)

**Platform:** [TryHackMe](https://tryhackme.com)  
**Category:** Network Forensics / PCAP Analysis
**Difficulty:** Easy

---

## Description

We received a short network capture from the guest network. The hint suggests very regular communication to `:8080`, suspicious headers, and hidden data. The goal is to identify the covert channel, rebuild the payload, decode it, and recover the flag.

---

## Step-by-Step Analysis (Command + Output)

### Step 1 — Isolate HTTP requests going to port 8080

```bash
tshark -r traffic.pcapng -Y "http && tcp.dstport==8080" -T fields -e frame.number -e ip.src -e ip.dst -e http.host -e http.request.method -e http.request.uri -e http.user_agent
```

**Output (excerpt):**

```text
16   192.168.1.141  34.41.103.191  byte-lotus-hotel.thm:8080  GET  /temp/updates.py  Mozilla/... Chrome/149...
391  192.168.1.141  34.41.103.191  byte-lotus-hotel.thm:8080  GET  /                  Mozilla/... ByteLotusClient/1.1
428  192.168.1.141  34.41.103.191  byte-lotus-hotel.thm:8080  GET  /                  Mozilla/... ByteLotusClient/1.1
520  192.168.1.141  34.41.103.191  byte-lotus-hotel.thm:8080  GET  /                  Mozilla/... ByteLotusClient/1.1
...
1300 192.168.1.141  34.41.103.191  byte-lotus-hotel.thm:8080  GET  /                  Mozilla/... ByteLotusClient/1.1
```

**Explanation:**
1. Frame 16 fetches `/temp/updates.py` (likely a stager/update script).
2. Then many regular `GET /` requests are sent to `:8080` with `ByteLotusClient/1.1`.
3. The URI is always `/`, so the hidden data is probably in headers.

---

### Step 2 — Print full request headers

```bash
tshark -r traffic.pcapng -Y "http.request && tcp.dstport==8080" -T fields -e frame.number -e http.request.line
```

**Output (excerpt):**

```text
391 ... Cookie: hotel_sess_state=HA==
428 ... Cookie: hotel_sess_state=AA==
520 ... Cookie: hotel_sess_state=BQ==
585 ... Cookie: hotel_sess_state=Mw==
619 ... Cookie: hotel_sess_state=Hg==
...
1300 ... Cookie: hotel_sess_state=NQ==
```

**Explanation:**
- The covert channel is inside the `Cookie` header.
- `hotel_sess_state` changes every request.
- Values look like short Base64 tokens (`HA==`, `AA==`, ...), likely one byte per request.

---

### Step 3 — Verify one token decode (`HA==` → `0x1c`)

```bash
echo 'HA==' | base64 -d | xxd -p
```

**Output:**

```text
1c
```

**Explanation:**
- `HA==` is Base64 text.
- After decode, it becomes raw byte `0x1c`.
- So each cookie value carries one raw byte.

---

### Step 4 — Manual XOR key discovery

TryHackMe flags usually start with `THM{`.
Use the first observed byte (`0x1c`) and expected first character (`T` = `0x54`):

- `0x1c ^ 0x54 = 0x48` → candidate key is `0x48`

Quick validation with next bytes:
- `0x00 ^ 0x48 = 0x48` (`H`)
- `0x05 ^ 0x48 = 0x4d` (`M`)
- `0x33 ^ 0x48 = 0x7b` (`{`)

**Explanation:** The key is consistently `0x48`, so payload = Base64-decoded bytes XOR `0x48`.

---

### Step 5 — Automate reconstruction with a Python script (`.py`)

Create `day4_decode.py` with this code:

```python
#!/usr/bin/env python3
import argparse
import base64
import re
import subprocess
import sys


def extract_cookie_values(pcap_path: str, cookie_name: str) -> list[str]:
    cmd = [
        "tshark", "-r", pcap_path,
        "-Y", "http.request && tcp.dstport==8080 && http.cookie",
        "-T", "fields", "-e", "http.cookie"
    ]

    out = subprocess.check_output(cmd, text=True, errors="ignore")
    values = []
    pattern = re.compile(rf"{re.escape(cookie_name)}=([^;\\s]+)")

    for line in out.splitlines():
        m = pattern.search(line)
        if m:
            values.append(m.group(1))
    return values


def decode_payload(values: list[str], xor_key: int) -> str:
    raw = b"".join(base64.b64decode(v) for v in values)
    decoded = bytes(b ^ xor_key for b in raw)
    return decoded.decode("utf-8", errors="replace")


def main() -> None:
    parser = argparse.ArgumentParser(description="Decode Day 4 covert cookie channel")
    parser.add_argument("pcap", help="Path to traffic.pcapng")
    parser.add_argument("--cookie", default="hotel_sess_state")
    parser.add_argument("--xor-key", default="0x48")
    args = parser.parse_args()

    try:
        xor_key = int(args.xor_key, 0)
    except ValueError:
        print("Invalid XOR key", file=sys.stderr)
        sys.exit(1)

    values = extract_cookie_values(args.pcap, args.cookie)
    if not values:
        print("No cookie chunks found", file=sys.stderr)
        sys.exit(1)

    print(f"[+] Extracted chunks: {len(values)}")
    print(decode_payload(values, xor_key))


if __name__ == "__main__":
    main()
```

Run it:

```bash
python3 day4_decode.py traffic.pcapng
```

**Output:**

```text
[+] Extracted chunks: 30
THM{[REDACTED]}
```

**Explanation:**
- The script extracts `hotel_sess_state` in frame order.
- It Base64-decodes each chunk.
- It XORs all bytes with `0x48`.
- The final output is the flag.

---

### Step 6 — Reproduce in CyberChef (no scripting)


1. `From Base64` (for each `hotel_sess_state` value in frame order)
2. Reassemble byte stream in order
3. `XOR` with key `48` (Hex)

**Recipe**
```text
From_Base64('A-Za-z0-9+/=',true,false)
XOR({'option':'Hex','string':'0x48'},'Standard',false)
```
**Input**
```text
HA==
AA==
BQ==
Mw==
...
```
**Output:**
```text
THM{[REDACTED]}
```

**Explanation:** CyberChef gives the same result as the Python script and is useful for visual validation.

---

## Command Option Breakdown

### `tshark`
- `-r traffic.pcapng` → read capture file
- `-Y "..."` → display filter
- `-T fields` + `-e <field>` → print selected fields only

### `base64 -d`
- `-d` → decode Base64 into raw bytes

### `xxd -p`
- `-p` → print bytes in plain hex

### `python3 day4_decode.py traffic.pcapng`
- `python3` → run the Python interpreter
- `day4_decode.py` → decoder script file
- `traffic.pcapng` → input capture file argument

---

## Root Cause

A covert client repeatedly exfiltrates data by embedding encoded byte chunks in an HTTP cookie (`hotel_sess_state`). Because the traffic pattern looks like normal polling and each payload is tiny, it blends into legitimate traffic unless headers are inspected and decoded.

---

## Result

Challenge solved by:
1. finding regular HTTP beaconing to `:8080`,
2. locating hidden chunks in cookie headers,
3. reconstructing Base64 byte chunks in frame order,
4. reversing XOR obfuscation.
