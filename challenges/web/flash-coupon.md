---
title: flash-coupon
category: web
difficulty: beginner
points: 100-500
author: h1tl4r
date: 2026-11-07
---
# flash-coupon &mdash; organizer write-up

> **Spoiler warning:** this file explains the vulnerability and the solution.
> Do not show it to players.

## Summary

FlashMart is a small Flask/SQLite shop. Every shopper starts with 10 credits.
The promo code `FLASH50` is supposed to add 50 credits once per shopper, and the
Flag Pack costs 200 credits. The flag is only rendered after a successful
purchase.

`POST /redeem` has a **time-of-check to time-of-use (TOCTOU)** bug: it checks
whether the coupon was already used, sleeps for 50 ms, and only then updates the
balance and the "redeemed" flag. The two steps are not atomic and are not
protected by a lock or a transaction. Sending many `/redeem` requests at the
same time makes all of them pass the check before any of them writes, so the
coupon is credited many times.

`POST /buy` is deliberately correct: a single `UPDATE ... WHERE balance >= 200`
checks and deducts atomically. The only vulnerability is `/redeem`.

The flag is `ingehack{ch3ck_th3n_us3_th3n_l0s3}`.

## Root cause

`src/app.py`, `redeem()`:

```python
# VULN: TOCTOU. This CHECK and the USE further down are two separate
# steps with no lock, transaction, or conditional UPDATE around them.
# Parallel requests all read "not redeemed yet" and then all add credits.
already_redeemed = conn.execute(
    "SELECT flash50_redeemed FROM users WHERE id = ?", (user["id"],)
).fetchone()["flash50_redeemed"]

if already_redeemed:
    return shop_page(user, error="You have already used the FLASH50 coupon.")

# Simulated backend work: fraud check, audit logging, partner API call.
time.sleep(0.05)

conn.execute(
    "UPDATE users SET balance = balance + ?, flash50_redeemed = 1 "
    "WHERE id = ?",
    (COUPON_VALUE, user["id"]),
)
```

Two properties make this exploitable:

1. **Check and use are separate statements.** SQLite commits each statement
   (the connection uses `isolation_level=None`), and every HTTP request opens
   its own connection. Nothing turns "read the flag, decide, write" into one
   protected operation.
2. **There is a 50 ms sleep between them.** The comment calls it fraud checking
   and audit logging, which sounds plausible in a real app. It also makes the
   window very wide: every request that arrives during those 50 ms still sees
   `flash50_redeemed = 0` and goes on to add 50 credits.

The server runs one gunicorn worker with the `gthread` worker class and 16
threads, so up to 16 requests really do execute inside that window at once.
SQLite serializes the writes, but each waiting writer has already passed the
check and still performs `balance = balance + 50`. Nothing rechecks the flag in
the `UPDATE`, which is the missing condition:

```sql
-- What the vulnerable update does (no re-check):
UPDATE users SET balance = balance + 50, flash50_redeemed = 1 WHERE id = ?;

-- What it should do (re-check inside the same statement):
UPDATE users SET balance = balance + 50, flash50_redeemed = 1
WHERE id = ? AND flash50_redeemed = 0;
```

With a starting balance of 10 and +50 per successful redemption, four
successful redemptions give 210 credits, which is enough for the 200-credit
Flag Pack. A single sequential redemption gives only 60, so normal use never
reaches the price.

## Exploitation

### Step 1: get a shopper session

Open the challenge URL (`http://localhost:8005/` with the local harness).
FlashMart creates a shopper automatically and sets the signed `session` cookie.
The page shows:

```
Your balance
10 credits
```

### Step 2: redeem the coupon sequentially

Enter `FLASH50` and submit. The balance becomes 60 and the coupon status becomes
"already used". Submitting `FLASH50` again is refused with "You have already
used the FLASH50 coupon." and the balance stays at 60. That is the correct,
non-race behaviour, and it still holds in the vulnerable build: only requests
that enter the check window concurrently can redeem twice.

### Step 3: send many redemptions in parallel

The trick is to send roughly 30 `POST /redeem` requests with
`code=FLASH50` at the same instant, reusing the same session cookie. Several of
them will pass the check during the 50 ms sleep and each adds 50 credits.

#### Option A: Burp Suite Repeater ("send group in parallel")

1. Turn on the Burp proxy and browse to the challenge so Burp sees the
   `GET /` request and the session cookie.
2. Send `POST /redeem` to Repeater (change the method if you sent `GET /`):
   ```
   POST /redeem HTTP/1.1
   Host: <host>
   Content-Type: application/x-www-form-urlencoded
   Cookie: session=<your session cookie>

   code=FLASH50
   ```
3. Duplicate the tab about 30 times (right-click the tab, "Duplicate tab").
4. Select all the tabs, right-click one of them, and choose
   **Send group in parallel** (in newer Burp versions this may be called
   **Send group in parallel (single-packet attack)**). If you do not see it,
   use Turbo Intruder below.
5. Refresh `GET /` in the browser. The balance should be well above 200 (often
   several hundred), and the flag can now be bought.

#### Option B: Turbo Intruder

Send the `POST /redeem` request to Turbo Intruder and run:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=1,
                           pipeline=False)
    for i in range(30):
        engine.queue(target.req, gate='race')
    engine.openGate('race')

def handleResponse(req, interesting):
    table.add(req)
```

`openGate` releases all 30 requests together. Check the balance with `GET /`
afterwards.

#### Option C: Python

The bundled solver does exactly this:

```bash
python3 solution/solve.py http://localhost:8005
```

It creates a fresh shopper, fires 30 threads through a `threading.Barrier` at
`POST /redeem`, reads the balance from `GET /`, and only then calls
`POST /buy` with `item=flag`. If a particular race does not produce enough
credits, it starts another fresh account and retries (up to 5 attempts).
Progress goes to stderr; stdout contains only the flag. The URL argument is
optional and defaults to `http://localhost:8005`.

A simple curl check of the normal behaviour (on a fresh account) is:

```bash
curl -c jar -b jar -s http://localhost:8005/ >/dev/null
curl -c jar -b jar -s -d 'code=FLASH50' http://localhost:8005/redeem >/dev/null
curl -c jar -b jar -s -d 'code=FLASH50' http://localhost:8005/redeem | grep -o 'already used'
```

### Step 4: buy the Flag Pack

Once the balance is at least 200, submit the "Buy Flag Pack" form (or
`POST /buy` with `item=flag`). The response contains the flag:

```
ingehack{ch3ck_th3n_us3_th3n_l0s3}
```

The flag is never stored in the database and is not rendered on any page until
`/buy` succeeds.

## Why the race is reliable here

- The window is 50 ms, which is huge compared with the time needed to read one
  SQLite row.
- gunicorn runs one worker with `--threads 16`, so 16 requests can be inside
  the window at once. A burst of 30 normally produces far more than the four
  successful redemptions needed.
- The check is a plain indexed lookup by primary key and does not block other
  readers. In WAL mode, readers do not block writers.
- A per-request SQLite connection and a 5-second busy timeout keep the
  serialized writes from failing with "database is locked"; they simply queue.
- The solver was run against five freshly built containers and printed the flag
  on all five attempts (5/5). It also retries with a new account if an
  unusually unlucky race yields fewer than 200 credits.

## The fix

Make the check and the update one atomic operation. The simplest fix is a
conditional `UPDATE` and a `rowcount` check:

```python
cursor = conn.execute(
    """
    UPDATE users
       SET balance = balance + ?,
           flash50_redeemed = 1
     WHERE id = ? AND flash50_redeemed = 0
    """,
    (COUPON_VALUE, user["id"]),
)

if cursor.rowcount == 0:
    return shop_page(user, error="You have already used the FLASH50 coupon.")

# Only the request that changed the row credits the balance.
```

Because SQLite evaluates the `WHERE` clause and performs the write as one
statement, exactly one concurrent request can match `flash50_redeemed = 0`.
Every other request sees `rowcount == 0` and is refused. No sleep, lock or
application-level check can reopen the window.

Equivalent fixes:

- **A transaction with an immediate write lock.** Use `BEGIN IMMEDIATE`,
  re-read the coupon state inside the transaction, update, and `COMMIT`. This
  serializes the whole check-then-use sequence. It is correct, but a single
  conditional `UPDATE` is simpler and faster.
- **A separate redemptions table with a unique constraint.** For example
  `redemptions(user_id, code)` with `UNIQUE(user_id, code)`. `INSERT` the
  redemption and let the unique constraint reject duplicates, then update the
  balance in the same transaction. Catch the integrity error and return
  "already used". This also gives you a useful audit trail.
- **Optimistic concurrency.** Read a version number and update
  `WHERE version = ?`, retrying or refusing if no row was changed.

`/buy` already demonstrates the correct pattern:
`UPDATE users SET balance = balance - 200 WHERE id = ? AND balance >= 200` and
then a `rowcount` check. The comparison and deduction happen in one statement,
so two simultaneous purchases cannot both spend the same credits.

## Files

- `chall.yaml` &mdash; platform manifest (name, scoring, description, flag)
- `src/app.py` &mdash; Flask app, SQLite schema and the vulnerable `/redeem`
- `src/templates/index.html`, `src/static/style.css` &mdash; FlashMart UI
- `Dockerfile` &mdash; Python 3.12, gunicorn gthread with 16 threads on 8080
- `solution/solve.py` &mdash; thread/barrier solver
