---
title: search-and-destroy
category: web
difficulty: beginner
points: 100-500
author: h1tl4r
date: 2026-11-07
---
# solution — search-and-destroy (organizer only)

**Flag:** `ingehack{qu0tes_br34k_qu3r13s}`

## Vulnerability

SQL injection (SQLi) in the book search. The `q` parameter is concatenated
directly into the SQL string, so the attacker controls part of the query and
can rewrite its `WHERE` clause. This is the only vulnerability in the app: all
other database access is parameterized, and there is no flag/debug route.

## Root cause

`GET /search?q=...` builds its query like this (`src/app.py`):

```python
q = request.args.get("q", "")

# VULN: user input is concatenated straight into the SQL query.
sql = (
    "SELECT id, title, author, price FROM books "
    "WHERE hidden = 0 AND title LIKE '%" + q + "%'"
)
rows = _db.execute(sql).fetchall()
```

The table has 15 public books (`hidden = 0`) and one staff-only book with
`hidden = 1`. That hidden row's title contains the flag. The query is meant to
return only public rows whose title matches the search term. Because `q` is
placed inside the SQL text instead of being bound as a value, the database
cannot tell the search term apart from SQL syntax.

## Why the quote causes an error

Searching for only a quote (`q='`) produces:

```sql
SELECT id, title, author, price FROM books
WHERE hidden = 0 AND title LIKE '%'%'
```

The first `'%'` is a valid string literal; the stray `%` and `'` are nonsense
to the parser, so SQLite raises a syntax error. The app deliberately shows the
short driver error in a red box so a beginner can see that their input reached
the database. The message contains no file paths.

## How the logic changes

The query ends with `... LIKE '%<q>%'`, and `AND` binds tighter than `OR`.
Injecting an always-true condition and commenting out the trailing `%'` turns
the clause into:

```sql
SELECT id, title, author, price FROM books
WHERE hidden = 0 AND title LIKE '%' OR 1=1-- -%'
```

SQLite treats `-- ` (two dashes followed by whitespace) as a comment to the end
of the line, so everything after it is discarded. The effective condition is:

```sql
WHERE (hidden = 0 AND title LIKE '%') OR 1=1
```

`OR 1=1` is true for every row, so the `hidden = 0` filter no longer matters and
the staff-only row is returned with the public catalogue:

```
Staff Preview - ingehack{qu0tes_br34k_qu3r13s}
```

## Exact payload

```
' OR 1=1-- -
```

URL-encoded by the browser or `requests` as needed:

```
http://<host>/search?q=%27%20OR%201%3D1--%20-
```

Public challenge URL (shared server):
`http://search-and-destroy.challs.ingeniums.club:9487/`
(or `https://search-and-destroy.challs.ingeniums.club:9488/`).

## Automated check

From the challenge folder, with the container running on host port 8002:

```bash
python3 solution/solve.py http://localhost:8002
```

The solver verifies the normal search does not leak the flag, sends the
payload, and prints only the flag found in the results HTML.

## Fix

Never build SQL by string concatenation. Use a parameterized query so the
driver sends the user input as data, not syntax:

```python
@app.get("/search")
def search():
    q = request.args.get("q", "")

    # Safe: the driver binds q as a value; it can never change the query.
    sql = (
        "SELECT id, title, author, price FROM books "
        "WHERE hidden = 0 AND title LIKE ?"
    )
    with _db_lock:
        rows = [
            dict(row)
            for row in _db.execute(sql, ("%" + q + "%",)).fetchall()
        ]

    return render_template("index.html", q=q, rows=rows, error=None)
```

Additional hardening:

- Keep the `hidden = 0` filter, but also avoid storing secrets in rows a public
  search can reach. The flag should not live in the table the search reads.
- Do not show raw database errors to users; log them server-side and return a
  generic message.
- Apply least privilege to the database account (read-only where possible).
