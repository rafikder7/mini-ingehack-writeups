---
title: ask-her
category: misc
difficulty: TBD
points: 100-500
author: boul3h14
date: 2026-11-12
---
# Solution — Ask Her

**Flag:** printed by the solver. The flag is owned by the platform admin UI
and is baked into the capture's encrypted record, so the value below the
walkthrough matches this deployment only.

The challenge is a four-stage recovery: repair enough of a ZIP to get the
backup tool, pull three XOR-obfuscated fragments out of three different
layers, rebuild the TLS master secret held in the memory image, then decrypt
the capture.

`solution/solve.py` automates the whole path and prints the flag.

## Step 0 — Get the handouts

The challenge ships four files as attachments:

`backup_corrupted.zip`, `memory.raw`, `traffic.pcap`, `README_challenge.txt`

They are also committed in the challenge's `files/` directory, so you can work
straight from the repository.

```bash
python3 solution/solve.py files/
```

The method is the same for everyone; the archive bytes, the memory image, the
capture and the flag are fixed for this deployment.

---

## Step 1 — Read the corrupted ZIP (local headers only)

The central directory and EOCD are gone (`unzip -l` fails), but the four local
file headers (`PK\x03\x04`) survive. Walk the file from header to header and
decompress each entry with `zlib.decompress(data, -15)`:

```python
i = raw.find(b"PK\x03\x04", off)
_ver, _flags, method, _t, _d, _crc, csize, _usize, nlen, elen = \
    struct.unpack("<HHHHHIIIHH", raw[i+4:i+30])
name = raw[i+30:i+30+nlen].decode()
data = raw[i+30+nlen+elen:i+30+nlen+elen+csize]
if method == 8:
    data = zlib.decompress(data, -15)
```

This yields `README.txt`, `backup_tool`, `notes.png` and `session.dat`.

## Step 2 — Three fragments

**A — `backup_tool --keygen`.** Without `BACKUP_KEY_A` it prints the decoy
`4b33`; with the variable set (any value) it prints the real
`Key fragment A: 4b32`. The strings are XOR-0x5a-obfuscated in `.data`, so
`strings` shows nothing.

**B — `session.dat`.** Scan the first 256 bytes for a 4-byte window that
de-XORs with `0x5a` to lowercase hex. The token sits at offset 64 and is
random per team (for example `9a7f`).

**C — `notes.png` alpha channel.** Pixel `(512,700)` holds the first byte and
`(513,700)` the second, both XORed with `0x5a`. The two bytes spell the hex
string in ASCII (for example `0xc0 0xde` → `c0de`), and both are random per
team. (The low-contrast QR is a decoy.)

## Step 3 — Rebuild the master secret from `memory.raw`

Extract the client random from the pcap and search the 16 MiB image for its
first 8 bytes — the marker sits at offset `0x2000`. The layout from there is:

| Offset | Contents |
|---|---|
| `0x2000` | 8-byte marker (first 8 bytes of the TLS ClientHello random) |
| `0x2008` | 48-byte master secret, XORed with a keystream |
| `0x2038` | CRC32(master secret), little endian |

The keystream is `SHA256(A + B + C)` (the ASCII hex strings — for example
`4b32` + `9a7f` + `c0de`), repeated to 48 bytes. XOR the 48 bytes at
`0x2008`, then confirm `zlib.crc32(secret) == crc32` read from `0x2038`.

## Step 4 — Decrypt the TLS session

`traffic.pcap` is TLS 1.2 with `TLS_RSA_WITH_AES_128_CBC_SHA` (`0x002f`).
Now that the master secret is known, derive the server keys with the TLS 1.2
PRF:

```
key_block = PRF(master_secret, "key expansion",
                server_random + client_random, 72)
server_write_MAC_key = key_block[20:40]   # SHA1
server_write_key     = key_block[56:72]   # AES-128
```

The session negotiated **RFC 7366 Encrypt-then-MAC** (extension `0x0016`),
so a repaired record body is laid out as:

```
explicit_IV (16) || AES-128-CBC ciphertext || MAC (20, SHA1)
```

The server's application-data record is the first encrypted record after its
`ChangeCipherSpec` (sequence number 1, after the Finished). Verify the MAC
over the *encrypted* fragment (MAC input =
`seq ‖ type ‖ version ‖ len(fragment) ‖ fragment`, where
`fragment = IV || ciphertext`), then decrypt the ciphertext, strip the TLS
padding, and the HTTP response contains the flag.

### Wireshark / tshark route

```bash
printf 'CLIENT_RANDOM %s %s\n' <client_random_hex> <master_secret_hex> > keylog.txt
tshark -r traffic.pcap -o tls.keylog_file:keylog.txt \
       -Y http.response -T fields -e http.file_data
```

In Wireshark: *Preferences → Protocols → TLS → (Pre)-Master-Secret log
filename*.

## Run the reference solver

```bash
python3 solution/solve.py files/
# [*] evidence directory: /path/to/files
# [*] recovered 4 entries from the corrupted zip
# [*] keygen: 'Key fragment A: 4b33' without env, 'Key fragment A: 4b32' with BACKUP_KEY_A
# [*] PNG alpha fragments -> <fragment C>
# [*] session.dat candidates: ['<fragment B>']
# [*] marker found at offset 0x2000
# [*] fragments A=4b32 B=<fragment B> C=<fragment C>; master secret recovered (CRC32 ok)
# ingehack{...}
```

Requirements: Python 3 and the `openssl` CLI (used for AES-CBC).

## Regenerating the handouts

The handouts are built by `src/generate.py` (authoring tool, no runtime
container). If the flag changes, rebuild them so the capture still decrypts to
the new value:

```bash
python3 src/generate.py --flag 'ingehack{...}' --out files
```

It rewrites `backup_corrupted.zip`, `memory.raw`, `traffic.pcap` and
`README_challenge.txt` with fresh fragments B and C, keeping them consistent
with each other.
