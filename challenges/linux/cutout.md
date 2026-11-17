---
title: cutout
category: linux
difficulty: hard
points: 500
author: boul3h14
date: 2026-11-11
---
# Cutout — solution (spoilers)

Chain of custody, step by step. Each head below can be pasted into a live
instance; the flights of the puzzle stay exactly as shipped.

**You are boul3h14.**

## 1 — Triage

Two dead-ends sit in `/opt/vendor-tool` (a toy binary and a conf that says
the key is not there). The real trail hides in the *split key* the vendor
tool ships next to its library:

```
/opt/vendor-tool/lib/.part1    -> 1st line "v3rm1l10n" + ledger pointer
/opt/vendor-tool/share/.part2  -> 1st line "0bs1d14n"  + "the offset is carved there"
/var/lib/vendor-tool/.ledger   -> <label> <offset> <purpose>, one row per process
```

The ledger is world-readable BY DESIGN: it reveals the offsets (1024 vs
2048) and names exactly one label as the **key** holder. Labels are random
per build; they appear in the six ch41n.py daemons' argv:

```
ps aux | grep ch41n.py
```

## 2 — The unlock ladder

Each daemon polls `/tmp/<label>.trigger` every 100 ms. On a match with its
unlock code it fchmods its fd to 0644 for 10 s and hands the fd to a keeper
running as *your* uid — that's the only way the kernel lets a cross-uid
reader in. Holder pids sit in `/run/cutout/<label>.holder`, so:

| step | code | who accepts | what you get |
|------|------|-------------|--------------|
| `start` | public (`/tmp/p1.unlock`) | P1 (the only label that takes it) | fd3 → `ingehack{` |
| (P1's plaintext) | `ingehack{` | P2+P3 (decoys), P4, P6 | `ingehack{f4k3_` ×2, 4096-byte blobs ×2 |
| (P4's plaintext) | `ch41n_` | P5 (the label that REJECTED `ingehack{`) | 4096-byte blob |

Read the holder's blob with python (the watchdog reaps naive `cat /proc/*/fd/3`):

```python
open(f"/proc/{holder_pid}/fd/3", "rb").read()
```

## 3 — Decrypt the blobs

XOR is cyclic under each blob's outer key:

```python
def blob2payload(b, offset, length, key):
    return bytes(x ^ key[i % len(key)] for i, x in enumerate(b[offset:offset+length]))

combined = b"v3rm1l10n0bs1d14n"                   # from the /opt split key
real_key = blob2payload(p6_blob, 2048, 14, combined)  # -> r34l_k3y_0bs1d14n
p4_frag  = blob2payload(p4_blob, 1024, 6,  real_key)  # -> ch41n_
p5_frag  = blob2payload(p5_blob, 1024, 11, real_key)  # -> 0f_cust0dy}
```

Both decoys decrypt to `ingehack{f4k3_` — two liars, as promised.

## 4 — Assemble

```
ingehack{ + ch41n_ + 0f_cust0dy}
```

Automated end-to-end check: `solutions/verify_solution.sh` (run as boul3h14)
prints `FLAG: ...` and exits 0.
