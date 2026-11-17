---
title: misplaced-keys
category: misc
difficulty: beginner
points: 100-500
author: s4l1
date: 2026-11-04
---
# Solution — Misplaced Keys

**Flag:** `ingehack{wr0ng_l4y0ut_r1ght_m3ss4g3}`

## Handout

| File | Purpose |
|---|---|
| `chat.txt` | Chat export. Both of `s4l1`'s messages are garbled; the teammate's reply is not |

## Idea

The keyboard sends the right physical keys, but the system interprets them
with the wrong layout: the person types on **QWERTY**, the text is
interpreted as **Dvorak**. Every character is replaced by the character that
sits on the same physical key in the other layout. It is a plain
monoalphabetic substitution, so there is no key to find.

## Recognising it

- The garbled text contains punctuation where letters should be (`.`, `,`,
  `?`, `+`, `{`), which a normal substitution of letters would not produce.
- `a`, `m`, digits and spaces stay as they are, matching the Dvorak/QWERTY
  overlap.
- The known flag prefix gives a crib: `ingehack{` must map to `cbi.dajt?`.
  Checking it against the Dvorak layout confirms the guess:

| intended (QWERTY key) | `i` | `n` | `g` | `e` | `h` | `a` | `c` | `k` | `{` |
|---|---|---|---|---|---|---|---|---|---|
| sent (Dvorak) | `c` | `b` | `i` | `.` | `d` | `a` | `j` | `t` | `?` |

## Decoding

Any "keyboard layout change" tool works (QWERTY ← Dvorak). Or script it by
mapping each received character to the key at the same position in QWERTY:

```python
Q  = "`1234567890-=" "qwertyuiop[]\\" "asdfghjkl;'" "zxcvbnm,./"
QS = "~!@#$%^&*()_+" "QWERTYUIOP{}|" "ASDFGHJKL:\"" "ZXCVBNM<>?"
D  = "`1234567890[]" "',.pyfgcrl/=\\" "aoeuidhtns-" ";qjkxbmwvz"
DS = "~!@#$%^&*(){}" "\"<>PYFGCRL?+|" "AOEUIDHTNS_" ":QJKXBMWVZ"

dec = {d: q for d, q in list(zip(D, Q)) + list(zip(DS, QS))}
msg = open("chat.txt").read()
print("".join(dec.get(c, c) for c in msg))
```

Note that the script also touches the timestamps and the teammate's line, so
read only the `s4l1` lines of the output.

First message decoded:

```
hey, i finally got it. the flag is ingehack{wr0ng_l4y0ut_r1ght_m3ss4g3} do not lose it
```

Second message decoded (flavour only):

```
my keyboard is acting up again. i will try on another one tomorrow
```

## Skills

| Skill | Used for |
|---|---|
| Recognising a keyboard-layout substitution | Identifying why punctuation appears inside words |
| Using a known plaintext (crib) | Confirming the layout from `ingehack{` |
| Simple scripting / online decoders | Reversing the mapping |

## Author notes

- `_` is typed as `Shift + -`, which Dvorak reads as `{`; the digits are
  identical in both layouts, so `0`, `3`, `4`, `1` survive unchanged.
- Generated programmatically from the layout tables above and decoded back to
  the original to verify the round trip.
