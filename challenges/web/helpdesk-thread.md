---
title: Open Ticket
category: web
difficulty: hard
points: 100-500
author: hxuu
date: 2026-11-06
---
# Solution — Open Ticket

Static container. The support agent runs on the platform's admin-bot worker
(`admin-bot.ts` in this directory, attached through `adminBot:` in `chall.yaml`).

## Intended path

1. The thread accepts a message body and renders it back to the player and to the agent.
2. The source shows the agent's view contains the flag, and that the agent's page sets a
   CSRF token cookie readable by script (the agent guidelines page frames the same thing
   as a convenience feature).
3. Store a payload. The support tool sanitises nothing on the agent's side. The payload
   carries the claim code printed on the player's ticket.
4. Post the message, then press the run-bot form on the challenge page: the worker
   signs the agent in with her password — which is the team's flag, attached to the job
   at run time and never present in the (public) handler source — and opens the desk
   view of the thread. The payload reads the token out of `document.cookie` and posts
   the flag into the player's own thread under that claim code.
5. Reload the thread as the player.

The desk files appended replies under the claim code, not under the shared student
account. The container stays static, but one team's flag no longer appears on another
team's ticket.

## Proof the flag is reachable

The flag appears in the player's thread only after the agent's visit. Direct requests
for the agent's view return nothing to a student session, and signing in as `agent`
needs her password — the flag itself. The session key, student pass, and desk token
are minted randomly at container start and never appear in `/src`.

## Verified (2026-11-06, local containers)

`solution/solve.py --stage prepare` stores the payload in the thread, a browser signed
in as the agent opened `/agent/tickets/1`, and `--stage submit` read the flag out of
the whole thread player-side. The full two-stage run is the `challenge-review` load:
prepare → run the bot (platform admin-bot, or locally `victim/visit.py` with `FLAG` in
the environment) → submit.

## Acceptance

A playtester with source and devtools gets the agent's browser to act within 25
minutes. This is the hardest single flag in the Roster; if it lands under 15 minutes
the sanitisation is doing too little work.

## Notes

- The agent guidelines page is the Diegetic Hint: it lists "tokens we keep in cookies
  so the front end can read them", presented as convenience, not as a risk.
- Deliberately non-HttpOnly cookie; that is the lesson, so it must look like a normal
  decision in the source.
- The thread is bounded so one visit cannot make the next one unloadable: request
  bodies over 16 KiB are rejected, message bodies are capped at 2,000 characters,
  and only the newest 500 messages (plus the two seeded ones) are kept. A payload
  that echoes the page back is therefore truncated on append, not multiplied.

## Running it end to end

`solution/solve.py --stage all --base-url <challenge URL> --platform-url
https://ctf.ingeniums.club --token <player token>` does the whole path: prepare, submit
the URL to the platform admin bot, wait for the visit, then read the flag. The
`--stage prepare` and `--stage submit` halves remain for `challenge-review`; pass the
claim code printed by prepare to submit with `--claim <code>`.

### One command

From the repository root, the unified runner needs only the platform id and a
player token:

`RCTF_TOKEN=<team token> python3 challenges/web/solve.py open-ticket`

It resolves this directory, uses `https://helpdesk-thread.challs.ingeniums.club:9488/`,
runs prepare, submits to the platform admin bot, waits for the visit, and prints the
flag. The full challenge URL works in place of the id:
`https://ctf.ingeniums.club/challenges?challenge=open-ticket`.
