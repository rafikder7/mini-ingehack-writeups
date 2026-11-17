---
title: Is this actually pwn?
category: pwn
difficulty: beginner
points: 100-500
author: derrm
date: 2026-11-01
---
# Solution — Is this actually pwn? (EsiMedia)

## Intended path

1. **Start the instance** on the platform and open its URL (an `http` expose,
   so browse it as `http://is-this-actually-pwn-<uid>.challs.ingeniums.club:<tcp-mapped-port>`).
   There is no Files section — the `crop_image` binary is fetched through the
   app's LFI (`/serve_file?filename=profile_pics/crop_image`).
2. **Analyze the binary** (`checksec`): No PIE, no stack canary, NX on.
   `main` reads two lines with `gets()` into `input_file` (rbp-0x210) and
   `output_file` (rbp-0x110), so the second `gets` reaches the saved RIP at
   offset **280**. The binary embeds `pop rax/rdi/rsi/rdx; ret` and
   `syscall; ret` gadgets (0x40162f–0x401638).
3. **Upload stage 2** via `/upload_temp_image` (any bytes; the app stores it
   as `instance/temp_<ts>_<name>`). Stage 2 is the string
   `/bin/bash\0-c\0cat /proc/1/environ>/app/instance/temp_flag.txt\0` followed
   by the argv pointer array `[bss, bss+10, bss+13, 0]` with `bss = 0x404100`.
   (The ROP `execve` passes `envp=0`, so the shell's own env is empty — the
   flag must be read from the app process, PID 1, via `/proc/1/environ`.)
4. **Trigger the overflow** via `/crop_and_save` with JSON
   `filename = "<temp_name>\0" + "A"*pad + ROP`, `x=y=0`, `width=height=100`.
   The app pipes six lines to the binary's stdin; line 2 is
   `/app/instance/out_cropped_<filename>`, so `pad = 280 - 26 - len(temp_name) - 1`.
   The `\0` makes `fopen` on line 1 open just the uploaded stage-2 file
   (fd 3), while `gets` on line 2 still stores the whole payload.
   Payload bytes must be `< 0x80` and `!= 0x0a` because the app pipes stdin
   with `text=True` (UTF-8) — that's why `bss = 0x404100`, not `0x4048e0`
   (0xe0 would be mangled to 0xc3 0xa0).
5. **ROP**: `read(3, 0x404100, 0x100)` pulls stage 2 from the uploaded file,
   then `execve("/bin/bash", ["/bin/bash", "-c", "cat /proc/1/environ>..."], 0)`.
   `MagickReadImage` fails on the fake image first, so `main` returns into
   the chain.
6. **Exfiltrate the flag**: the flag is the instance's `RCTF_FLAG` env var
   (per-team, baked at launch) in the app process (PID 1). Read it back
   through the LFI: `/serve_file?filename=temp_flag.txt` (`temp_` prefix →
   instance dir). `/proc` is blocked in `/serve_file`, so the env is only
   reachable via the shell.

## Solver

```sh
python3 solution/solve.py http://is-this-actually-pwn-<uid>.challs.ingeniums.club:<port>
# locally: docker run -p 5000:5000 -e RCTF_FLAG=ingehack{local_test} <image>
#          python3 solution/solve.py http://127.0.0.1:5000
```

`solve.py` performs steps 3–6 automatically and prints the flag.

## Verified locally

- `docker build` of the challenge Dockerfile succeeds.
- Container run with `RCTF_FLAG=ingehack{local_test_flag_123}`; `solve.py`
  against it prints `ingehack{local_test_flag_123}`.
- `/serve_file?filename=/proc/self/environ` → 404 (blocklist works);
  `/serve_file?filename=profile_pics/crop_image` → 200 (LFI works).
