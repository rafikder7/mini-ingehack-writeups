---
title: rumor-mill
category: linux
difficulty: easy
points: 100-500
author: boul3h14
date: 2026-11-03
---
# Rumor Mill — intended solution

Connect over the TLS-wrapped SSH route (TTY required):

```sh
ssh -o ProxyCommand='openssl s_client -quiet -servername %h -connect %h:%p' \
    y4ss1n3@rumor-mill.challs.ingeniums.club -p 9450
# password: Welcome123
```

Locally (plain Docker port publish):

```sh
ssh -p 2222 y4ss1n3@localhost
```

## 1. The handover note

```sh
cat ~/onboarding.txt
```

It names `n0d4` and gives his password: `K4r4k0_1962`.

```sh
su - n0d4
# password: K4r4k0_1962
```

## 2. The split password

```sh
cd ~
cat .part1        # h4y   (no trailing newline)
cat .part2        # 3l1y4 (no trailing newline)
cat README        # boul3h14's password = .part1 + "_" + .part2
```

Join them literally:

```
h4y + _ + 3l1y4 = h4y_3l1y4
```

```sh
su - boul3h14
# password: h4y_3l1y4
```

## 3. The flag

```sh
cat ~/flag.txt
```

```
ingehack{su_1s_n0t_4lm1ghty_y0u_kn0w}
```

## 4. The decoy

`gh3nj4` is the lie. `boul3h14`'s note points at the decoy; a copy lives in
`boul3h14`'s own home (ACL fallback, see the challenge README):

```sh
cat ~/gh3nj4_flag_copy.txt
# ingehack{th1s_1s_4_f4k3_fl4g}
```

Submitting the decoy is wrong. Only `boul3h14`'s `flag.txt` is real.

## Why `su` and not `nc`

`su` uses PAM and reads the password from the controlling terminal. A raw
`nc` session has no TTY, so the chain cannot be completed that way.

## One-shot solver

```sh
CONTAINER=rumor-mill solution/verify_solution.sh
```

The script runs every step as `y4ss1n3`, derives `boul3h14`'s password from
the two fragments, prints the real flag, and exits 0.
