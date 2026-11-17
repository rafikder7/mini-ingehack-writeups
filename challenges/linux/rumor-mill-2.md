---
title: rumor-mill-2
category: linux
difficulty: medium
points: 100-500
author: boul3h14
date: 2026-11-03
---
# Rumor Mill #2 — The Encoded Rumor: Solution Walkthrough

## Overview
- **Category:** Linux
- **Difficulty:** Medium (200–500 pts)
- **Flag format:** `ingehack{...}`

## Step-by-Step Solution

### Step 1: SSH as y4ss1n3
The deployed route is wrapped by Traefik's TLS terminator (`tcp-ssl` port
9450):

```bash
ssh -o ProxyCommand='openssl s_client -quiet -servername %h -connect %h:%p' \
    y4ss1n3@rumor-mill-2.challs.ingeniums.club -p 9450
# Password: Welcome123
```

Locally (plain Docker publish):

```bash
ssh y4ss1n3@localhost -p 2225
# Password: Welcome123
```

### Step 2: Read onboarding.txt
```bash
cat onboarding.txt
```
You'll see a base64-encoded password for n0d4: `SzRyNGswXzE5NjI=`

### Step 3: Decode the base64 — but it's stale
```bash
echo SzRyNGswXzE5NjI= | base64 -d
# Output: K4r4k0_1962
```
Try to switch to n0d4:
```bash
su n0d4
# Password: K4r4k0_1962
# → Authentication failure!
```
The password has been rotated. The onboarding note is stale.

### Step 4: Check .bash_history
```bash
cat .bash_history
```
The history shows that the real password is in `.env`:
```
# the note is stale, real one is in .env
cat .env
```

### Step 5: Read .env
```bash
cat .env
```
Output:
```
# n0d4 password rotated 2026-11-01
N0D4_B64=SzRyNGswXzE5NjJfdjI=
```

### Step 6: Decode the rotated password
```bash
echo SzRyNGswXzE5NjJfdjI= | base64 -d
# Output: K4r4k0_1962_v2
```

### Step 7: Switch to n0d4
```bash
su n0d4
# Password: K4r4k0_1962_v2
# → Success!
```

### Step 8: Read n0d4's README
```bash
cat README
```
Output:
```
boul3h14's password = join(decode(.part<order[0]>), decode(.part<order[1]>),
decode(.part<order[2]>)) with "_"

All parts are base64. The order is base64-encoded in .part_order.
There is a .part4 — it is a decoy. Do not use it.
```

### Step 9: Read and decode .part_order
```bash
cat .part_order
# Output: MiwzLDE=

echo MiwzLDE= | base64 -d
# Output: 2,3,1
```
The order is: part2, part3, part1.

### Step 10: Decode the parts in order
```bash
cat .part2.b64
# Output: aDR5
echo aDR5 | base64 -d
# Output: h4y

cat .part3.b64
# Output: M2wxeTQ=
echo M2wxeTQ= | base64 -d
# Output: 3l1y4

cat .part1.b64
# Output: d3M0eQ==
echo d3M0eQ== | base64 -d
# Output: ws4y
```

### Step 11: Join parts in order 2,3,1 with underscores
```
h4y_3l1y4_ws4y
```

### Step 12: Switch to boul3h14
```bash
su boul3h14
# Password: h4y_3l1y4_ws4y
# → Success!
```

### Step 13: Read the encrypted flag
```bash
cat flag.txt
# → Encrypted blob (not the flag)

cat README
```
The README tells you the flag is encrypted with a key in `.key.b64`.

### Step 14: Decode the key
```bash
cat .key.b64
# Output: bWF0ZGlyZXNoX2VsXzNyYWI=

echo bWF0ZGlyZXNoX2VsXzNyYWI= | base64 -d
# Output: matdiresh_el_3rab
```

### Step 15: Decrypt the flag
```bash
openssl enc -d -aes-256-cbc -a -pbkdf2 -in flag.txt -k matdiresh_el_3rab
```

### Flag
```
ingehack{rum0rs_4r3_b4s364_4ll_th3_w4y_d0wn}
```

## Traps and Decoys
1. **Stale password in onboarding.txt** — `K4r4k0_1962` is the OLD password. The real one is in `.env`.
2. **`.part4.b64` is a decoy** — decodes to `n0p3`. Do not use it.
3. **gh3nj4 has a fake flag** — `ingehack{th1s_1s_4_f4k3_fl4g}` is NOT the real flag.
4. **flag.txt is encrypted** — you cannot just `cat` it. You need the OpenSSL decryption key.
