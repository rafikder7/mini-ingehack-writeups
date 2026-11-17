---
title: El Minha
category: web
difficulty: medium
points: 100-500
author: hxuu
date: 2026-11-06
---
# Solution — Bursary

Static container. The bursar runs on the platform's admin-bot worker
(`admin-bot.ts` in this directory, attached through `adminBot:` in `chall.yaml`).

## Intended path

1. Sign in as `student` (the pass is printed on the login page) and read
   `/statement`: a balance, a claim code, and no refund.
2. The footer links a public sandbox. Publish a page there; it comes back at a stable
   address on the same site.
3. Put a form in the page that posts `amount`, `studentId=student`, and your claim code
   to `/refund`, and add a line that submits the form as soon as the page loads.
4. Submit that sandbox address through the run-bot form on the challenge page. The
   worker signs the bursar in with her password — which is the team's flag, attached
   to the job at run time and never present in the (public) handler source — and then
   opens the link. The request carries her session, and the console approves it.
5. Reload the student statement: the credit is there and so is the flag.

The help page explains that transfers are protected because "the session cookie is
signed". That is true, and it says nothing about who caused the request.

The refund is filed under the claim code printed on the player's statement, not under
the shared student account. The container stays static, but one team's refund no longer
appears on another team's statement.

## Proof the flag is reachable

The intended path lands a refund under the player's claim code and `/statement` renders
the flag afterwards. A direct `POST /refund` from the student's own session is refused
with 403 and the statement still shows no refund. Sign-ins as `bursar` need her
password, which is the flag: nobody signs in as her without already having solved it,
and the session key, student pass, and every other credential are minted randomly at
container start and never appear in `/src`.

## Verified (2026-11-06, local containers)

`solution/solve.py --stage prepare` publishes the page, a browser signed in as the
bursar opened the sandbox link, and `--stage submit` read the flag. The full two-stage
run is the `challenge-review` load: prepare → run the bot (platform admin-bot, or
locally `victim/visit.py` with `FLAG` in the environment) → submit.

## Acceptance

A beginner builds the auto-submitting page from devtools and gets the bursar's refund
into their statement within 25 minutes, and can say why the bursar's browser was the one
that sent it.

## Running it end to end

`solution/solve.py --stage all --base-url <challenge URL> --platform-url
https://ctf.ingeniums.club --token <player token>` does the whole path: prepare, submit
the URL to the platform admin bot, wait for the visit, then read the flag. The
`--stage prepare` and `--stage submit` halves remain for `challenge-review`; pass the
claim code printed by prepare to submit with `--claim <code>`.

### One command

From the repository root, the unified runner needs only the platform id and a
player token:

`RCTF_TOKEN=<team token> python3 challenges/web/solve.py el-minha`

It resolves this directory, uses `https://campus-refund.challs.ingeniums.club:9488/`,
runs prepare, submits to the platform admin bot, waits for the visit, and prints the
flag. The full challenge URL works in place of the id:
`https://ctf.ingeniums.club/challenges?challenge=el-minha`.
