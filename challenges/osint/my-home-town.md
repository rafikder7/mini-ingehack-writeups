---
title: MyHomeTown
category: osint
difficulty: hard
points: 300-500
author: RedhaIWNL
date: 2026-11-09
---
# Solution — MyHomeTown

**Flag:** `ingehack{PELISSIER_PALMYRA_10_35.233_5.702}`

## Answers

| Part | Answer | Why |
|---|---|---|
| `SURNAME` | `PELISSIER` | The 1862 plaque names the “Duc de Malakoff, Maréchal de France”; the man behind that title is Aimable-Jean-Jacques Pélissier. The flag drops the accent. |
| `CITY` | `PALMYRA` | The Roman site at El Kantara (ancient *Calceus Herculis*) was garrisoned by a *numerus Palmyrenorum* — auxiliary archers from Palmyra. |
| `SPAN` | `10` | The Roman bridge has a single stone arch with a 10 m diameter/span. |
| `LAT_LON` | `35.233, 5.702` | The camera point is `35.2332394, 5.7019108`; WGS84, latitude first, rounded to three decimals. |

## Intended path

1. **Find the site.** The handout is a view over a palm-filled gorge toward a
   small building on a ridge. The three research questions point to a Roman
   bridge in Algeria: the plaque, the Palmyrene auxiliary archers, and the
   single arch. Search combinations such as *Roman bridge Algeria single arch
   plaque Duc de Malakoff* and *Palmyrene archers Calceus Herculis*. The site is
   El Kantara, Biskra, the Roman *Calceus Herculis*.

2. **The plaque → `PELISSIER`.** The bridge was restored in 1862 by the French
   military engineers. The commemorative plaque reads:

   > “1862 restauré par le génie militaire, Napoléon III Empereur et Duc de
   > Malakoff Maréchal de France, Gouverneur Général de l'Algérie.”

   “Duc de Malakoff” was the victory title granted to Marshal
   Aimable-Jean-Jacques Pélissier. Flag value: `PELISSIER` (no accent).

3. **The archers → `PALMYRA`.** The Roman garrison at *Calceus Herculis*
   included Palmyrene auxiliary archers (*numerus Palmyrenorum*). The city is
   Palmyra. Flag value: `PALMYRA`.

4. **The arch → `10`.** The bridge is a single-arch stone bridge; the arch is
   quoted as 10 m in diameter. Flag value: `10`.

5. **The camera point → `35.233, 5.702`.** The author’s camera position is
   `35.2332394, 5.7019108`, on the public road/bridge approach on RN 3. The
   small building on the ridge is the visual anchor, not the requested point.
   Round the camera point to three decimal places:

   - latitude `35.2332394` → `35.233`
   - longitude `5.7019108` → `5.702`

   Flag value: `35.233_5.702`.

6. **Join the parts** in `SURNAME_CITY_SPAN_LAT_LON` order:

   ```text
   ingehack{PELISSIER_PALMYRA_10_35.233_5.702}
   ```

## Sources

- Plaque text, Roman bridge, single 10 m arch, *Calceus Herculis*:
  <https://les-souvenirs-de-claude.e-monsite.com/pages/le-voyage/hotel-el-kaid-et-le-site-d-el-kantara.html>
- Palmyrene archers at *Calceus Herculis* / El Kantara:
  <https://hal.science/hal-04602244v1/document>
  and <https://www.cambridge.org/core/books/palmyra-the-mediterranean-and-beyond/you-cant-get-there-from-here/F0D79D5F3965D3FBE9886CF0FB8A5501>
- Aimable-Jean-Jacques Pélissier, 1st Duc de Malakoff:
  <https://en.wikipedia.org/wiki/Aimable_P%C3%A9lissier>
  and <https://www.britannica.com/biography/Aimable-Jean-Jacques-Pelissier-duc-de-Malakoff>

## Author notes

- The handout is the only file; the plaque is not included. Players must find
  the bridge and the plaque from the questions and the landscape.
- The image has been stripped of metadata; it contains no GPS, timestamp,
  camera, or source path.
- The requested coordinate is the camera position, not the small building.
  Keep that distinction if the challenge is updated.
- The small building is a real structure. Do not add a closer or more
  identifying photo of it, and do not publish a private-residence location.
