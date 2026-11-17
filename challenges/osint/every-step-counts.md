---
title: every-step-counts
category: osint
difficulty: TBD
points: 100-500
author: s4l1
date: 2026-10-24
---
# Solution — Every Step Counts

**Flag:** `ingehack{Strava_Nrg8000_27-01-2018_James_Quarles_Polar}`

All five facts were checked against primary or major-press sources.

---

## Q1 — `Strava`

Search: `fitness app heatmap military bases 2017`. Strava published its Global
Heatmap in November 2017.

## Q2 — `Nrg8000`

Nathan Ruser, ANU international security student. His tweet:
https://x.com/Nrg8000/status/957318498102865920
Bellingcat's Strava guide (29 Jan 2018) names him as the first to notice.

## Q3 — `27-01-2018`

The tweet's timestamp is 2018-01-27 18:24 UTC (Ruser is in Australia, where that
moment is already the 28th — hence the "(UTC)" in the question). Press embeds all
read "January 27, 2018".
Author bonus: a tweet ID encodes its timestamp (snowflake). ID 957318498102865920
→ 2018-01-27 18:24:50 UTC.

## Q4 — `James_Quarles`

Search: `Strava CEO open letter January 29 2018`. TechCrunch quotes the open letter
from CEO James Quarles; the 14 Feb 2018 Senate letter (Coons/Flake) refers to
"On January 29, 2018, you issued a response".

## Q5 — `Polar`

Bellingcat + De Correspondent, 8 July 2018: "After Strava, Polar is Revealing the
Homes of Soldiers and Spies"
https://www.bellingcat.com/news/2018/07/08/strava-polar-revealing-homes-soldiers-spies/
(company: Polar Electro, Finland; product: Polar Flow). Answer is the brand, `Polar`.

---

## Skills

| Q | Skill |
|---|---|
| 1 | Event/keyword search |
| 2 | Linking news coverage to a social profile, exact username |
| 3 | Reading timestamps, timezone awareness |
| 4 | Corporate/executive lookup, cross-checking press with official records |
| 5 | `site:` dorking, following an investigative follow-up |
