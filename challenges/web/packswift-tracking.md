---
title: Where Requests End
category: web
difficulty: hard
points: 100-500
author: hxuu
date: 2026-11-06
---
# Solution — Where Requests End

Instanced. No Victim. Series A, part 3. Capstone.

## Intended path

1. The tracking page works normally. The edge in front of it terminates chunked bodies
   and rejects duplicate length headers.
2. The Deploy Log records the edge hardening, and in the same breath that the
   application behind it "still prefers a length header and does not read chunking".
3. Write a request the two components read differently: the edge sees one request and
   forwards the body, the application sees a length that ends early and treats the
   remainder as a second request.
4. The second request is one nobody sent — it reaches the role endpoint behind the edge,
   which trusts its caller.
5. The response carries the flag.

## The crafted request

One request, one length header, chunked body:

```
POST /track HTTP/1.1
Host: t
Content-Length: 15
Transfer-Encoding: chunked

64
id=PS-1042&GET /internal/role HTTP/1.1
Host: t
X-PackSwift-Role: dispatcher
Connection: close

0

```

The chunk size (hex) is `64` = 100 = 11 bytes of filler plus the 89-byte smuggled
request. The length header is `15` = the two chunk-size characters, their CRLF, and the
eleven filler bytes. The application reads 15 bytes (the chunk line and the filler),
then parses the next bytes as a new request. The edge never saw that request, so its
`/internal/` rule and its role-header strip do not apply.

## Why the instance is exposed as tcp-ssl

An `http` expose puts Traefik's HTTP router in front of the edge. That router parses
the request itself: it drops one of two length headers and re-frames a chunked body
before forwarding, so the edge and the application never see a disagreement. A
`tcp-ssl` expose terminates TLS and forwards the stream byte for byte, which keeps
the crafted request intact. This was verified on the live instance: the same bytes
that return the tracking page over `http` return no flag, and over `tcp-ssl` they
reach the role endpoint.

## Proof the flag is reachable

`solution/solve.py` writes the raw request once, reads until the edge closes, prints
exactly the flag, and exits 0. It uses sockets, not an HTTP client, because a client
would normalise the request. The endpoint is TLS, so the socket is wrapped with
`ssl` and the host name is passed as SNI before the bytes go out.

Well-formed requests to `/internal/role` are answered 403 by the edge. A duplicate
length header is answered 400 by the edge. A smuggled role request without the role
header is refused by the application. From a cold container, only the crafted request
yields the flag.

## Acceptance

A playtester who has been told the challenge is hard, and who may ask for help, reaches
the privileged endpoint within 90 minutes. Setup must take under 10 of those minutes
using the onboarding page at `/proxy-setup`.

## Open items

- Static flag on an instanced challenge: sharing is possible. Accepted for a beginner
  event; revisit if it becomes a scoreboard problem.

## Running it end to end

`solution/solve.py --platform-url https://ctf.ingeniums.club --token <player token>`
starts the team's instance if it is stopped, waits for the endpoint, wraps the socket
in TLS, and prints the flag. `--base-url <instance URL>` skips the instance step.

### One command

From the repository root, the unified runner needs only the platform id and a
player token:

`RCTF_TOKEN=<team token> python3 challenges/web/solve.py packswift-tracking`

It starts the team's instance, solves it, and prints the flag. Add
`--stop-instance` to stop the instance afterwards. The full challenge URL works in
place of the id: `https://ctf.ingeniums.club/challenges?challenge=packswift-tracking`.
