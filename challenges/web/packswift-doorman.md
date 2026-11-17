---
title: The Doorman
category: web
difficulty: beginner
points: 100-500
author: hxuu
date: 2026-11-06
---
# Solution — The Doorman

Static container. No Victim. Series A, part 1.

## Intended path

1. Open the portal. There is a public tracking form and a "staff" area that refuses you.
2. Read the page footer, which links to the portal's own source.
3. The source shows the role coming from a request header, and `/status` confirms it
   is present.
4. Open devtools → Network → the portal request → Edit and Resend → add the header the
   source names → send.
5. The dispatcher dashboard renders, and the flag is on it.

## Proof the flag is reachable

A single request carrying the header returns 200 on `/portal/dispatch` and the body
contains the flag. No other route does.

## Acceptance

A playtester who has never opened devtools finds the dispatcher dashboard within 25
minutes, without being told the header name.

## Running it

`python3 challenges/web/solve.py the-doorman`

The unified runner resolves this directory and runs its solver against
`https://packswift-doorman.challs.ingeniums.club:9488/`. No token is needed for
this static challenge. The full challenge URL works in place of the id:
`https://ctf.ingeniums.club/challenges?challenge=the-doorman`.

## Open items

- `/status` names the `X-PackSwift-Role` header and says the gateway set it upstream,
  without printing its value.
- The footer of every page links to `/src`, which renders `app.py` with line numbers
  and no flag.
