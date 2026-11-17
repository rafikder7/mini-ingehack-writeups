---
title: rumor-mill-3
category: linux
difficulty: hard
points: 100-500
author: boul3h14
date: 2026-11-03
---
# Rumor Mill #3 — The Liars' Club: intended solution

## Overview

- **Category:** Linux
- **Difficulty:** Hard
- **Points:** 500
- **Flag format:** `ingehack{...}`
- **Connection:** `ssh y4ss1n3@<HOST> -p 2222` (password `4rt4z4`, TTY required)

Every note lies. The auth log does not. The chain is:

```
y4ss1n3 ──(nohup.out)──> n0d4 ──(auth.log)──> boul3h14 ──(.sync)──> flag
             ▲                ▲                    ▲
        onboarding lie    README lie          confession lie
```

---

## 1. Start as `y4ss1n3`

```bash
ssh y4ss1n3@<HOST> -p 2222
# password: 4rt4z4
```

Read the onboarding note:

```bash
cat ~/onboarding.txt
# bWF0ZGlyZXNo
echo bWF0ZGlyZXNo | base64 -d
# matdiresh
su - n0d4
# Password: matdiresh
# su: Authentication failure
```

`onboarding.txt` is stale and the log says so. Now read the process log:

```bash
cat ~/nohup.out
# [2026-11-02T03:14:23Z] new password: 1ng3_d4wl4
```

```bash
su - n0d4
# Password: 1ng3_d4wl4
```

## 2. As `n0d4`: the README lies about order

```bash
cat ~/README
# boul3h14's password = decode(.part2) + decode(.part1)
# No separator. If .part3 exists, ignore it.

cat ~/.part1.b64    # OWF0Y2hhbA==  -> 9atchal
cat ~/.part2.b64    # bHdlbGQ=      -> lweld
cat ~/.part3.b64    # bjBwMw==      -> n0p3 (ignore)
```

Following the README gives `lweld9atchal`:

```bash
su - boul3h14
# Password: lweld9atchal
# su: Authentication failure
```

The README lies about both the **separator** and the **order**. The machine
does not lie:

```bash
grep -n 'printf' /var/log/rumor/auth.log
# 2026-11-02T10:12:44Z n0d4 EXEC: su -c 'printf "%s_%s" "9atchal" "lweld"' boul3h14
```

So the real password is `9atchal` + `_` + `lweld`:

```bash
su - boul3h14
# Password: 9atchal_lweld
```

## 3. As `boul3h14`: every note points somewhere else

```bash
cat ~/flag.txt
# the flag moved. check .confession

cat ~/.confession
# I left the real flag in gh3nj4's .cache at 10:45.
# The path is /home/gh3nj4/.cache/.flag_backup. Don't tell anyone.
```

That destination is a decoy:

```bash
cat /home/gh3nj4/.cache/.flag_backup
# cat: /home/gh3nj4/.cache/.flag_backup: Permission denied

find / -name '.flag_backup' 2>/dev/null
# (empty: gh3nj4's home is 0711, .cache is 0711, the file is 0600)
```

The recovery note is also a lie:

```bash
cat ~/.recovery.txt
# I forgot the key. Try the old one: matdiresh_key

openssl enc -d -aes-256-cbc -a -pbkdf2 \
    -k matdiresh_key -in ~/.sync/.backup.bin
# bad decrypt / garbage
```

The bash history only plants the fake:

```bash
cat ~/.bash_history
# echo 'ingehack{th1s_1s_4_f4k3_fl4g}' > /tmp/flag_test
# cp /root/real_flag.txt /home/gh3nj4/.cache/.flag_backup

cat ~/verify_gh3nj4.txt
# The string ingehack{th1s_1s_4_f4k3_fl4g} is the decoy.
```

And `audit_copy.log` is doctored:

```bash
grep 'sync' ~/audit_copy.log
# ... /home/n0d4/.sync/.backup.bin ...
# (never boul3h14/.sync: the copy was rewritten)
```

## 4. The real log names the real `.sync`

```bash
grep -E 'openssl|notes|real flag' /var/log/rumor/auth.log
# 2026-11-02T11:00:02Z admin openssl ... -out /home/boul3h14/.sync/.backup.bin
# 2026-11-02T11:00:03Z admin echo -n <redacted> | base64 > /home/boul3h14/.sync/.notes.txt
# 2026-11-02T11:01:02Z admin note: real flag only in boul3h14/.sync.
```

The directory is `0711 root:root`, so it cannot be listed, but a known
absolute path is readable:

```bash
cat ~/.sync/.notes.txt
# bHdlbGRfZWxfZmFyZjBzaA==
echo bHdlbGRfZWxfZmFyZjBzaA== | base64 -d
# lweld_el_farf0sh
```

The blob's strings header confirms the source and that the bytes are
encrypted:

```bash
strings ~/.sync/.backup.bin
# SYNCED_FROM:/home/gh3nj4/.cache/.flag_backup
# SYNCED_AT:2026-11-02T11:00:02Z

strings ~/.sync/.backup.bin | grep -F 'ingehack{'
# (empty)
```

## 5. Decrypt

```bash
openssl enc -d -aes-256-cbc -a -pbkdf2 \
    -in ~/.sync/.backup.bin \
    -k lweld_el_farf0sh
# ingehack{3v3ry0n3_l13d_bu7_th3_l0g_d1dn7}
```

## Why the shortcuts fail

- `onboarding.txt` gives `matdiresh`, which is not n0d4's password.
- The README's concatenation (`lweld9atchal`) and order are wrong.
- n0d4's `.sync` decrypts to random bytes with `matdiresh_key`.
- `gh3nj4/.cache/.flag_backup` is `0600 gh3nj4`, and `0711` on the parent
  blocks `find` and `ls`.
- `.confession` and `audit_copy.log` both point at `n0d4/.sync`.
- `.bash_history` and `verify_gh3nj4.txt` only contain the fake flag.
- The real `.backup.bin` is encrypted; `strings` and `cat` are useless.
- The real key exists only base64-encoded in `.sync/.notes.txt`.
- There is no sudo binary and no SUID/capability shortcut.

## Cleanup

The solve path is read-only. If you create test files, remove them:

```bash
rm -f /tmp/flag_test
```
