---
title: employee-profile
category: web
difficulty: beginner
points: 100-500
author: h1tl4r
date: 2026-11-07
---
# solution — employee-profile (organizer only)

**Flag:** `ingehack{n3v3r_trust_th3_1d_1n_th3_url}`

## Vulnerability

IDOR (insecure direct object reference), also called broken object-level
authorization. `/profile` checks that a user is logged in, but never checks
that the requested employee record belongs to that user. A logged-in guest can
read any employee's profile by changing the `id` in the URL; the CEO's profile
(`id=1`) contains the flag in its `notes` field.

This is the only vulnerability in the app. Sessions are properly signed,
passwords are stored as salted hashes, every SQL query is parameterized, and
the guest profile contains no secret.

## Root cause

`GET /profile?id=<n>` builds the page from whatever integer arrives in the
query string (`src/app.py`):

```python
@app.get("/profile")
def profile():
    if "user_id" not in session:
        return redirect(url_for("login"))

    requested_id = request.args.get("id", type=int)
    if requested_id is None:
        return redirect(url_for("profile", id=session["user_id"]))

    # VULN: only authentication is checked here. The code never verifies that
    # requested_id belongs to session["user_id"], so any logged-in employee can
    # read any other employee's profile by changing the number in the URL.
    employee = get_user_by_id(requested_id)
    if employee is None:
        abort(404)

    viewer = get_user_by_id(session["user_id"])
    return render_template("profile.html", employee=employee, viewer=viewer)
```

Authentication answers "is this person signed in?". Authorization answers "is
this person allowed to see **this object**?". The route only does the first
one. The database lookup itself is safe (`WHERE id = ?`), but it happily
returns any row the caller names.

## Exact solution

1. Open the login page (`/login`) on the challenge URL:
   `http://employee-profile.challs.ingeniums.club:9487/` (or the TLS form on
   `:9488`).
2. Sign in with the advertised demo account:
   - username `guest`
   - password `guest123`
3. The app redirects to your own profile: `/profile?id=105`.
4. The employee number is right there in the URL and is a plain integer.
   Change `105` to `1` (the CEO, `admin`): `/profile?id=1`.
5. The page now shows **Maya Chen, CEO**. Her private notes contain:

   ```
   ingehack{n3v3r_trust_th3_1d_1n_th3_url}
   ```

The IDs are sequential and visible, the guest's own id is `105`, and the CEO is
employee number `1`. No hashing, guessing or brute force is needed; enumerating
the directory (101–110) only shows harmless fake employee data.

## Automated check

From the challenge folder, with the container running on host port 8003:

```bash
python3 solution/solve.py http://localhost:8003
```

It confirms the logged-out and guest views do not leak the flag, logs in as
`guest`, requests `/profile?id=1`, and prints only the flag found in the HTML.

## Fix

Enforce object-level authorization on every request, after authentication and
before returning data. Compare the requested object with the identity in the
session:

```python
@app.get("/profile")
def profile():
    if "user_id" not in session:
        return redirect(url_for("login"))

    requested_id = request.args.get("id", type=int)
    if requested_id is None:
        return redirect(url_for("profile", id=session["user_id"]))

    # Authorization: employees may only read their own record.
    if requested_id != session["user_id"]:
        abort(403)

    employee = get_user_by_id(requested_id)
    if employee is None:
        abort(404)

    viewer = get_user_by_id(session["user_id"])
    return render_template("profile.html", employee=employee, viewer=viewer)
```

If the app has roles, load the role from the database/server-side session and
add the role check explicitly (for example, HR and admins may read other
profiles). Do not trust a `role` value supplied by the client.

Additional hardening:

- Apply the same ownership check to every route, API endpoint and download
  that takes an object id. One protected page does not protect the others.
- Prefer an authorization helper/middleware so a new route cannot forget the
  check (for example, `load_employee_or_403(id)`).
- Where object ids need not be sequential, use unguessable identifiers (UUIDs)
  as defense in depth. That is not a substitute for the authorization check: a
  leaked or guessed id must still be rejected.
- Return a generic `403 Forbidden` outside the caller's scope so the response
  does not reveal whether the object exists.
- Keep secrets out of ordinary profile fields that broad read paths can reach;
  a flag should live behind its own well-authorized route.
