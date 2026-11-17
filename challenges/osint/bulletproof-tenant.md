---
title: bulletproof-tenant
category: osint
difficulty: TBD
points: 100-500
author: s4l1
date: 2026-10-24
---
# Solution — Bulletproof Tenant

**Flag:** `ingehack{AS51852_Private_Layer_Switzerland_Panama_SSH_brute-force}`

All facts were verified on 2026-10-24 against live sources (ASN registries,
geolocation APIs, public blocklists, GreyNoise, Shodan InternetDB, and a direct
SSH banner grab).

---

## Q1 — `AS51852`

Look up `179.43.153.2` on any IP/ASN service (`ipinfo.io`, `ip-api.com`,
`bgp.he.net`, or an RIR RDAP endpoint). Every source returns the same origin:

- ASN: **51852**
- AS name: `PLI-AS`
- Announcement: `179.43.128.0/18`

## Q2 — `Private_Layer`

The ASN's registered organization is **Private Layer INC**. Dropping the legal
suffix gives `Private Layer`; spaces become underscores in the flag.
(Their own site confirms they operate AS51852 and have done so since 2010.)

## Q3 — `Switzerland`

The prefix is registered through LACNIC to a Panama entity, but the address
itself geolocates to **Switzerland** — Rümlang / Zurich (canton ZH), per
`ipinfo.io`, `ipwho.is`, `ip-api.com`, and `ip.sb`. The provider advertises
"multiple Switzerland based data center locations" and its first datacenter
opened there in 2010.

## Q4 — `Panama`

The RIR record is the giveaway. LACNIC RDAP for `179.43.153.2`
(`https://rdap.lacnic.net/rdap/ip/179.43.153.2`) lists the registrant as
**PRIVATE LAYER INC**, Suite 1404, Torres De Las Americas, Torre C, Panama
City, Panama.

The same country appears in the USTR's 2018 Notorious Markets list, in the
section **"PRIVATE LAYER-HOSTED SITES"**:

> "This group of websites is hosted by Private Layer, which reportedly is
> operated from Panama with data center and hosting operations in Switzerland
> and elsewhere. [...] this is the fifth consecutive year that the List has
> stressed the significant international trade impact of Private Layer's
> hosting services..."

(PDF: `https://torrentfreak.com/images/2018_Notorious_Markets_List.pdf`.)

## Q5 — `SSH_brute-force`

The IP is a long-standing SSH-abuse source. Confirmed by multiple independent
public feeds on the same day:

| Source | Result |
|---|---|
| `blocklist.de` API (`api.blocklist.de/api.php?ip=179.43.153.2&format=json`) | `{"attacks":"444","reports":"6"}` |
| `lists.blocklist.de/lists/ssh.txt` | IP present |
| OpenDBL `opendbl.net/lists/bruteforce.list` | IP present |
| FireHOL `bruteforceblocker.ipset` | IP present |
| FireHOL `blocklist_de_ssh.ipset` | IP present |
| `ipsum.txt` (stamparm) | score `10` blacklists |
| GreyNoise Community API | `classification: malicious`, `noise: true` |
| AbuseIPDB public reported-IP sitemap | IP indexed |

Direct banner grab on port 22 returns:

```
SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.18
```

Shodan InternetDB independently reports port `22`, CPE
`cpe:/a:openbsd:openssh:8.9p1`, and `cpe:/o:canonical:ubuntu_linux`.
So the box is a Linux SSH server being used as a brute-force launchpad.

---

## Author notes

- The two "country" answers are deliberately different: **Switzerland** is
  where the packets land, **Panama** is where the company is registered. The
  USTR quote in Q4 is the unambiguous tie-breaker.
- Counts like blocklist.de's `444` and the exact OpenSSH patch level are
  time-sensitive. They are evidence, not answers — do not put them in the flag.
- AbuseIPDB itself sits behind Cloudflare; players will see the reported-IP
  page in a normal browser. The challenge does not require its live score,
  only that the IP is on file there.
- Historical passive DNS for the address has included `academyclub.net`,
  `massive-network.com`, `technologyduo.com`, and `filmsegodnya.com` — useful
  flavour, not needed to solve.

## Skills

| Q | Skill |
|---|---|
| 1 | IP → ASN pivot with BGP/registry lookups |
| 2 | Reading ASN/WHOIS organization records |
| 3 | IP geolocation vs. corporate registration |
| 4 | RIR RDAP/whois + cross-checking a public regulatory filing |
| 5 | Reading threat-intel feeds, recognising brute-force indicators, banner grabbing |
