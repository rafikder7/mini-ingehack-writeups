---
title: Dispatch
category: web
difficulty: medium
points: 100-500
author: hxuu
date: 2026-11-06
---
# Solution — Dispatch

Static container. No Victim. Series A, part 2.

## Intended path

1. Ask for `/internal/dispatch`. The edge answers 403 with a hand-written page saying
   the path is not available to customers.
2. Read the Deploy Log (linked from the 403 page footer, in-world as "what changed this
   quarter"). It records the edge rule and notes the application "decodes URLs as the
   framework does by default".
3. The application decodes the path before routing, so an encoded slash reaches the same
   console the edge refuses.
4. Request the encoded form. The edge does not match its prefix rule; the application
   routes to dispatch and the flag renders.

## Proof the flag is reachable

One request returns the dispatch console body containing the flag, while the plain path
returns 403 from the edge.

## Acceptance

A playtester finds a path the edge rejects and the application accepts within 25
minutes, and can say which component made which decision.

## Running it

`python3 challenges/web/solve.py dispatch`

The unified runner resolves this directory and runs its solver against
`https://packswift-dispatch.challs.ingeniums.club:9488/`. No token is needed for
this static challenge. The full challenge URL works in place of the id:
`https://ctf.ingeniums.club/challenges?challenge=dispatch`.

## Open items

- `src/` not written yet. Two components in one container: an edge process with a prefix
  rule, and the application behind it.
- The encoded-slash behaviour must be deliberate in the app, not incidental to the
  framework, so the source explains it.
