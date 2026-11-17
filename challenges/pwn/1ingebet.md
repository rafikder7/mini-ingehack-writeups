---
title: 1ingeBet
category: pwn
difficulty: beginner
points: 100-500
author: derrm
date: 2026-11-03
---
# solution — 1ingeBet (static, shared instance)

Organizer-only writeup. The automated solver is `solve.mjs` in this directory:

```sh
node solve.mjs http://1ingebet.challs.ingeniums.club:9487
```

It prints the flag after winning 10 rounds in a row.

## Intended solve (step by step)

1. Open the site, click "read our certified source" in the footer (or DevTools →
   Sources) and read `/rng.js`, `/deck.js`, `/engine.js`. They are served
   non-minified on purpose, with no comments.
2. `rng.js`: the RNG is a 16-bit LCG —
   `state = (state * 1103515245 + 12345) mod 65536`, and
   `randInt(n) = floor(state / 65536 * n)` using the value *after* the update.
3. `engine.js`: round N uses seed `s_n = (seed_1 + (N-1) * 1337) mod 65536`
   (`SEED_STEP = 1337`), where `seed_1` is the one seed shown at session start
   (the server never sends it again).
4. `deck.js`: fixed deck order (suits ♠ ♥ ♦ ♣, ranks A..K) and Fisher–Yates
   top-down — for `i = 51 … 1`, `j = randInt(i+1)`, swap. One shuffle consumes
   exactly 51 RNG outputs.
5. The bug, in `engine.js`: after every shuffle, `deck.splice(0, 2)` — the first
   2 cards of the shuffled deck are silently discarded. Dealing then uses deck
   positions 2..5: `deck[2]` player card 1, `deck[3]` dealer up, `deck[4]` player
   card 2, `deck[5]` dealer hole, `deck[6..]` hits. A predictor that forgets the
   burn is offset by exactly 2 cards and fails.
6. Recompute each round's shuffle in a spreadsheet (or 20-line script) from
   `s_n`, read off the dealer's hole card before playing.
7. Play by hand: win the winnable rounds, press **Sit out** on the rest (the
   round advances, the streak is preserved). At **Streak 10/10** the response
   carries the flag.

## Spreadsheet formulas

Let `A1` hold the session seed shown at start. For round N:

- Round seed: `=MOD($A$1 + (N-1)*1337, 65536)`
- RNG chain (one row per output, starting from the round seed in `B1`):
  - `B1 = <round seed>`
  - `B2 = MOD(B1*1103515245 + 12345, 65536)` (fill down 51 rows per shuffle)
- Fisher–Yates, top-down: for step `i` (51 down to 1), the j used is
  `=INT(B<row>/65536 * (i+1))` where `B<row>` is the next unused RNG output.
  Swap `deck[i]` and `deck[j]` in the sheet's deck column (rows 1..52 in fixed
  order ♠A..♠K, ♥A..♥K, ♦A..♦K, ♣A..♣K).
- After the 51 swaps, **delete the first 2 rows** (the burn cards). Then:
  player card 1 = row 1, dealer up = row 2, player card 2 = row 3, dealer hole =
  row 4, hits from row 5 onward.

## Verification (challenge-review ladder, run locally before merge)

- `docker build .` in the challenge directory succeeds.
- Local run: `FLAG='ingehack{local_test}' PORT=8123 node src/server.js`, then
  `node solution/solve.mjs http://127.0.0.1:8123` → wins 10 in a row and prints
  `ingehack{local_test}`. Predicted hole card matched the actual hole card on
  every played round.
- API hygiene checks (all passing, 41/41 in the project's acceptance suite):
  seed appears only in the first `/api/session` response; `dealerHole`/
  `dealerCards`/`dealerTotal` are null during the player phase; `/flag.txt`,
  `/.env`, `/server.js` and traversal paths 404; `/rng.js` `/deck.js`
  `/engine.js` served byte-identical to the server's copies; out-of-phase or
  bogus actions rejected with 400/404 and state unchanged; ~3 actions/sec rate
  limit returns 429; sessions expire after 2 h.

## Platform notes

- Static challenge: one shared container behind the challs Traefik at
  `1ingebet.challs.ingeniums.club` (http :9487 / https :9488 — mapped ports
  rotate, re-check `docs/status.md` at ship time). The flag is injected as the
  `FLAG` env var by `ops/deploy-static.sh`; `src/server.js` reads `process.env.FLAG`
  first and falls back to a `flag.txt` file for local runs. The container listens
  on port 8080 (fixed convention).
- `hidden: true` in `chall.yaml` until publish — CI re-registration defaults
  `hidden` to false, so it must stay explicit.
