---
title: inspect-the-source
category: web
difficulty: beginner
points: 100-500
author: h1tl4r
date: 2026-11-07
---
# Challenge 1 &mdash; inspect-the-source (Organizer Only)

**Flag:** `ingehack{view_source_1s_y0ur_fr13nd}`

## Vulnerability

The "secret" admin credentials and two pieces of the flag are shipped to every
visitor in client-side HTML and JavaScript. Authentication is also enforced in
the browser, so the check can be read and skipped trivially.

This is the one intended path; the app has no other vulnerability.

## Root cause

Developers treated the browser as a trusted environment:

1. Build notes containing flag fragment 1/3 were left in an HTML comment.
2. `src/templates/app.js` (served as `/static/app.js`) hardcodes the admin
   username, the base64-encoded admin password, and flag fragment 2/3.
3. The server re-checks the same leaked credentials and returns fragment 3/3
   only on a successful login.

A frontend check is not a security control: the user controls the browser and
can always read, modify, or skip JavaScript. Anything sent to the browser is
public.

## Intended solution

Flag pieces (in order, for the real flag):

| Piece | Value | Where |
|-------|-------|-------|
| 1/3 | `ingehack{vie` | HTML comment on `/` |
| 2/3 | `w_source_1s_` | `/static/app.js` |
| 3/3 | `y0ur_fr13nd}` | response to a successful `POST /login` |

Concatenating them gives `ingehack{view_source_1s_y0ur_fr13nd}`.

### Step 1 &mdash; Find fragment 1/3 in the page source

1. Open `http://localhost:8001/` (or the public challenge URL).
2. Press `F12` (or `Ctrl+Shift+I`) and open the **Elements** tab, or press
   `Ctrl+U` to view the raw source.
3. At the top of `<body>` there is a comment:
   ```html
   <!--
     Vault Co. portal - build notes
     TODO: remove debug notes before launch
     flag fragment 1/3: ingehack{vie
   -->
   ```
4. Copy `ingehack{vie`.

### Step 2 &mdash; Read app.js and recover the credentials

1. In DevTools, open the **Sources** tab (or the **Network** tab, reload, and
   select `app.js`; the **Response** panel shows the file).
2. Select `/static/app.js`. It contains, in plain text:
   ```js
   const ADMIN_USER = "admin";
   const ADMIN_PASS_B64 = "dmF1bHQtYWRtaW4tMjAyNA==";
   const DEBUG_FLAG_FRAGMENT = "w_source_1s_";
   ```
3. Copy fragment 2/3: `w_source_1s_`.
4. Decode the password. In the DevTools **Console**:
   ```js
   atob("dmF1bHQtYWRtaW4tMjAyNA==")
   // "vault-admin-2024"
   ```
   Base64 is encoding, not encryption.

### Step 3 &mdash; Log in to get fragment 3/3

1. Use username `admin` and password `vault-admin-2024` on the login form.
2. The form's JavaScript passes its hardcoded check, submits to `/login`, and
   the server responds with:
   ```
   flag fragment 3/3: y0ur_fr13nd}
   ```
3. Copy `y0ur_fr13nd}`.

### Step 4 &mdash; Assemble the flag

```
ingehack{vie + w_source_1s_ + y0ur_fr13nd}  =>  ingehack{view_source_1s_y0ur_fr13nd}
```

Submit `ingehack{view_source_1s_y0ur_fr13nd}`.

### Automated check

From the challenge folder (`challenges/web/inspect-the-source`), with the
container running on host port 8001:

```bash
python3 solution/solve.py http://localhost:8001
```

The URL argument is optional and defaults to `http://localhost:8001`. The
solver fetches `/`, parses the comment, fetches `/static/app.js`, base64-decodes
the password, posts the credentials to `/login`, and prints only the assembled
flag on stdout (progress goes to stderr).

## Fix

- Never put flags, credentials, API keys, or internal TODOs in HTML, JS, source
  maps, or any other file sent to the browser. Treat everything the client
  receives as public.
- Perform authentication and authorization only on the server (as `/login`
  already does for the final check) and do not rely on client-side validation.
- Keep debug build notes out of production templates; strip comments and source
  maps from release builds.
- Rotate any credential that was ever shipped to a client.
