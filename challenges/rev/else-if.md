---
title: else-if
category: rev
difficulty: medium
points: 100-500
author: 4ym3n
date: 2026-11-05
---
# else-if

The handout is a stripped, dynamically linked x86-64 Linux ELF. The flag is
the input accepted by its custom stack virtual machine:

```text
1ngh4ck{100_l4y3rs_d33p_4nd_st1ll_just_4_vm}
```

## Run the solver

Python 3 and its standard library are sufficient. From the repository root:

```sh
python3 challenges/rev/else-if/solution/solve.py
```

To also execute the handout and verify the recovered input on x86-64 Linux:

```sh
python3 challenges/rev/else-if/solution/solve.py --verify
```

Expected output:

```text
1ngh4ck{100_l4y3rs_d33p_4nd_st1ll_just_4_vm}
validation passed
```

An optional positional argument selects a different copy of the handout.
The extraction itself does not execute the binary.

## Find the interpreter

Start with `file`, `strings`, and `objdump -d -M intel` on `files/else_if`.
The program takes one command-line argument. `main`, at RVA `0x10a0`,
passes that argument, its length, and a `0x1d33`-byte embedded program at
RVA/file offset `0x2140` to the interpreter at RVA `0x1200`.

The interpreter recognizes the magic `NVM1`, then dispatches byte-sized
opcodes through a jump table. Each program starts with a 20-byte header:

| Offset | Meaning |
| --- | --- |
| `0x00` | Magic `NVM1` |
| `0x05` | Byte used as the decryption seed |
| `0x08` | Code size, little-endian uint32 |
| `0x0c` | Encrypted child size, little-endian uint32 |
| `0x10` | Encrypted message size, little-endian uint32 |

The code, encrypted child, and encrypted message follow the header in that
order. Relevant opcodes are:

| Opcode | Operation |
| --- | --- |
| `0x01` | Push a uint32 immediate |
| `0x02` | Push the input byte at a uint16 index |
| `0x03` | Push the input length |
| `0x20` | Compare equal |
| `0x21` / `0x22` | Compare unsigned less-than / less-than-or-equal |
| `0x23` / `0x24` | Compare unsigned greater-than / greater-than-or-equal |
| `0x30` | Pop a condition; add a uint16 weight to the counter if true |
| `0x31` | Reject unless the counter equals a uint32 immediate |
| `0x40` | Pop a key and decrypt the child program |
| `0x41` | Recursively execute the decrypted child; push its result |
| `0x42` | Decrypt and print the success message using the candidate |
| `0xff` | Return whether the top stack value is nonzero; an empty stack succeeds |

## Decode the wrappers

The outer program's code is:

```text
01 52 00 00 00 40 41 ff
```

This pushes key `0x52`, decrypts a child, executes it, and returns its
result. The implementation of `0x40` reveals the transformation:

```python
child = bytes(
    value ^ ((key + 29 * seed + 13 * index) & 0xff)
    for index, value in enumerate(encrypted_child)
)[::-1]
```

The decoded child has another `NVM1` header and the same eight-byte wrapper
shape, with a new key and seed. Repeat until the code changes. There are
99 wrappers and one final validator: 100 program layers in total.

## Recover the input

The final validator has 4,665 bytes of code, no child, and an 18-byte
encrypted success message. It awards 15 points for an input length of 44.
For each byte, it includes four comparisons against the same target `t`:

```text
input[i] >= t    weight 2
input[i] <  t    weight 1
input[i] <= t    weight 2
input[i] >  t    weight 1
```

Together these contribute four points exactly when `input[i] == t`, and
three otherwise. Further bounds, arithmetic, and adjacent-byte checks add
at most 175 points. The required final counter is `0x16e`, or 366, which
is `15 + 44 * 4 + 175`. Every character must therefore match its target.

Extracting each `INPUT_BYTE index; PUSH32 target; GREATER_OR_EQUAL`
instruction sequence yields the flag. The first target is `0x31`, the
digit `1`; the prefix is intentionally `1ngh4ck`, as declared in the
player-facing description.

The recovered input prints `validation passed` and exits with status 0.
Changing the last byte to `!` prints `access denied` and exits with status
1. The manifest declares the accepted input as the fixed platform flag.
