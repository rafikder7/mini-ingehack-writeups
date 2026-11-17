---
title: stub-offline
category: rev
difficulty: hard
points: 100-500
author: 4ym3n
date: 2026-11-05
---
# stub-offline

From the challenge directory:

```sh
python3 solution/solve.py --verify
```

The token is split into chunks of 9, 9, 5, 16, 6 and 12 bytes. The Python
part loads a stripped shared library and passes each chunk to `check1`
through `check6`.

## Unpacking

Base64-decode `PAYLOAD`. Check `TAG` as HMAC-SHA256 of `NONCE || ciphertext`
using `KEY`. XOR each 32-byte block with:

```text
SHA256(KEY || NONCE || uint32_le(block_number))
```

Gunzip the result to get the ELF. The key is in the loader, so this part
can be copied straight into an extraction script.

## The first five checks

1. **XXTEA.** Three uint32 words, 23 rounds, delta `0x9e3779b9`. The data
   block holds a 16-byte key and a 12-byte target. Decrypt the target and
   drop the three trailing zero bytes: `l0ck3d_up`.
2. **Linear equations.** A 9x9 matrix and nine targets, all uint16 values.
   The check computes `M * input mod 257`. Gaussian elimination gives
   `5p4gh3tt1`.
3. **RC4.** The eight-byte key sits beside the five encrypted bytes.
   Decryption gives `c0mp1`.
4. **ELF header.** FNV-1a-64 of the first 64 bytes, XORed with an eight-byte
   salt. Pack the result little-endian and hex-encode it. This chunk changes
   if the native build's header changes. Uppercase hex is rejected.
5. **Bytecode.** Each input byte goes through XOR, rotate-left and addition,
   then gets compared against a constant. Undo them in reverse order to get
   `3lfhdr`.

The bytecode instructions are `11 i` (load), `22 k` (xor), `33 r` (rotate),
`55 b` (add), `44 t` (compare) and `ff` (halt). Operands are single bytes;
arithmetic wraps at 256. Halt requires all six indices to have been checked.

The solver locates each data block by its eight-byte tag and reads the
fields that follow it.

## Last check

The last block is a saved Diffie-Hellman exchange:

```text
tag[8] | p:u32 | g:u32 | A:u32 | B:u32 | nonce:u64 | ciphertext[16]
```

All integers are little-endian. Here `p = 2147483647`, `g = 7`, and the
client exponent is derived from the final 12 input bytes:

```text
a = 1 + FNV1a32(input) % (p - 2)
A = pow(g, a, p)
shared = pow(B, a, p)
```

The group is small enough for baby-step giant-step: about 46,341 table
entries. Recover `a` from `A`, then calculate the shared value using `B`.

The shared value becomes an XTEA key:

```text
[shared, shared ^ 0xa5c31f73, low32(nonce), high32(nonce)]
```

Encrypt `nonce` and `nonce + 1` with 32-round XTEA to get two CTR keystream
blocks. XOR the ciphertext to recover 12 bytes and their little-endian
CRC32. The result is `0ffl1n3_r4t3`. Recompute the exponent from these bytes
to confirm it matches. No requests are made; the exchange is in the library.

Concatenate the six chunks and wrap them in `ingehack{...}`.

## Building

Requires GCC for x86-64 Linux and Python 3.9+.

```sh
python3 solution/build.py
python3 solution/verify.py --rebuild
unshare --user --map-root-user --net python3 solution/verify.py
```

The build writes `files/stub.py` and updates the flag in `chall.yaml`.
Keep those two files together when rebuilding: a different compiler may
change the ELF fingerprint. The temporary header and shared library are
discarded. Players only get `files/stub.py`.

The checks cover the accepted flag, mutations at all 57 token positions,
bad lengths, damaged payloads and a repeat build. The `unshare` command
runs the checks in a fresh network namespace.
