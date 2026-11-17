---
title: sanity-check
category: sanity
difficulty: beginner
points: 0
author: hxuu
date: 2026-10-23
---
# solution — sanity-check (welcome challenge, static, 0 points)

Walkthrough, so review can verify the intended path:

1. Open the challenge; the URL is in the description:
   `http://sanity-check.challs.ingeniums.club:9487/` (plain HTTP).
2. The page welcomes the player and says the flag is on a different path of
   the same host.
3. Request `GET /flag` (address bar: append `/flag`).
4. Response body is `ingehack{http_is_just_requests}` (fixed; shared).
5. Submit it on the platform. The challenge pays 0 points, so it never moves
   the scoreboard — `tiebreakEligible: false` as well.

Solver one-liner:

```sh
curl -s "http://sanity-check.challs.ingeniums.club:9487/flag"
```
