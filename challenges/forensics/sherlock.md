---
title: sherlock
category: forensics
difficulty: TBD
points: 100-500
author: Ch1Hub
date: 2026-10-31
---
# Sherlock — Solution Walkthrough

## Overview
- **Category:** Forensics / OSINT / CTI
- **Author:** Ch1Hub
- **Flag:** `ingehack{30469_2025-10-21}`

## Intended Solve Path

1. **Identify the Organization:**
   From the description, identify the targeted victim organization:
   `Raleigh Family Practice` (certified Level 3 Patient-Centered Medical Home).

2. **Threat Intelligence / Leak Discovery:**
   - Search the web for recent data breaches, leaks, or ransomware posts concerning "Raleigh Family Practice".
   - Identify that the threat actor responsible for the breach is the **Booba Project** threat group / ransomware group.

3. **Locate Data Leak Site (DLS):**
   - Check ransomware tracking platforms like [Ransomlook](https://www.ransomlook.io/) for Booba Project's official Data Leak Site (DLS).
   - Find their Tor onion service: `http://7t3zi3e7ki6iseun77ofqtr6wmbpgnpc2ada6gstcxp54lw6q2zb7jad.onion/leaks`

4. **Access Leak & Download Archive:**
   - Using Tor Browser (or onion proxy routing), navigate to the Booba Project leaks section.
   - Locate the leaked archive for Raleigh Family Practice (`data.rar`, ~108MB).
   - Download and extract the archive.

5. **Examine the Records:**
   - Inspect the extracted directories and navigate to `Referrals/`.
   - Open `Referrals.xlsx`.
   - Switch to the sheet for year `2025` (as directed in the prompt: "According to the 2025 clinic records").

6. **Filter for Emma Martin:**
   - Search for patient `Emma Martin` (recorded as `Martin,Emma`).
   - Row 1715 contains the entry:
     - **Date:** `09/04/25`
     - **Name:** `Martin,Emma`
     - **Chart:** `30469`
     - **Specialist:** `Raleigh OB-already selected`
     - **A/I/P/UNC:** `Athena`
     - **Status of Referral:** `10/21/2025`

7. **Flag Construction:**
   - Chart Number = `30469`
   - Referral appointment date = `2025-10-21` (formatted as `YYYY-MM-DD`)
   - Flag format: `ingehack{ChartNumber_YYYY-MM-DD}`
   - Result: `ingehack{30469_2025-10-21}`
