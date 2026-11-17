---
title: ghost-steel
category: osint
difficulty: TBD
points: 100-500
author: s4l1
date: 2026-10-24
---
# Solution — Ghost Steel

**Flag:** `ingehack{SU-122-54_Dnepr_77_SU-100_M1968}`

Single source for Q2 and Q4 (and the lower bound for Q3):
Warford, "Armored Vehicle Development Behind the Curtain: the Secret Life of the
Soviet SU-122-54 Assault Gun", ARMOR, July–September 2016.
Live PDF: https://www.benning.army.mil/armor/eARMOR/content/issues/2016/JUL_SEP/3Warford16.pdf
(also indexed under moore.army.mil — the site changed domains Benning → Moore → Benning).
Wayback snapshot (verified 2026-10-24, 5 pages, text search confirms Q2/Q4):
http://web.archive.org/web/20250815071651/https://www.benning.army.mil/armor/eARMOR/content/issues/2016/JUL_SEP/3Warford16.pdf

---

## Q1 — `SU-122-54`

Reverse image search `ghost.png`, or describe it: "Soviet casemate tank destroyer
T-54 chassis". The photo shows several vehicles; the target is the foreground one
(hull number 211). It is a museum exhibit at Kubinka (Russia).

## Q2 — `Dnepr`

Warford: "In September 1967, the Soviet military launched Exercise Dnepr ...
Almost unnoticed, however, the Soviet army also deployed a previously unseen new
assault gun." Also on tanks-encyclopedia and the Exercise Dnepr wiki page.
Spelling is "Dnepr" in the English-language sources (not "Dnieper"), hence the
"5 letters" format hint.

## Q3 — `77`

| Source | Figure |
|---|---|
| tanks-encyclopedia.com | 77 built |
| wiki.warthunder.com | 77 |
| warhistory.org | 77 |
| wargame.fandom.com | 77 |
| armorama review | 77 |
| Warford (ARMOR 2016) | "between 77 and 95 ... exact number still unknown" |
| keymilitary.com | "not exceeded 100" |

The question asks for the figure most sites give → 77. The lesson: even the best
primary analysis calls the number uncertain.

## Q4 — `SU-100_M1968`

1. Dork: `"SU-122-54" Warford filetype:pdf` → the ARMOR article (PDF).
2. Read it. Passage: the vehicle was "misidentified as the 'SU-100 M1968'" in the
   formerly top-secret US report "Soviet Ground Force Weapons and Armored
   Vehicles" (August 1969).
3. Space → underscore: `SU-100_M1968`.

Not on any wiki — players must open and read the PDF.

---

## Author notes

- The PDF is the single external dependency. Check it loads shortly before the event.
- Old blog links to this PDF are dead because they contain a typo (`JUL_SEp`),
  use http, and lowercase `earmor`; the working path is `eARMOR/.../JUL_SEP/`.
- Run the Q4 dork once in Google before launch; it was verified with a different
  search engine.

## Skills

| Q | Skill |
|---|---|
| 1 | Reverse image search, picking the right object in a busy photo |
| 2 | Plain search, spelling/transliteration awareness |
| 3 | Cross-checking sources, noticing when sources disagree |
| 4 | `filetype:` dorking + reading a primary source for an unwikied fact |
