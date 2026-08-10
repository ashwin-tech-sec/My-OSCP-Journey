# Poison — Medium — FreeBSD

**Date:** 08 - August - 2026

---
## Tags

`#freebsd` `#web` `#php` `#lfi` `#password-reuse` `#vnc` `#port-forwarding` `#privesc`

---

## What I Know

| Item       | Detail                                                                          |
| ---------- | ------------------------------------------------------------------------------- |
| Target     | 10.129.1.254 (host: Poison)                                                      |
| OS         | FreeBSD 11.1-RELEASE                                                             |
| Open ports | 22, 80                                                                           |
| Services   | OpenSSH 7.2 (22); Apache 2.4.29 / PHP 5.6.32 (80)                                |
| Foothold   | LFI in `browse.php?file=` → read `pwdbackup.txt` → base64 ×13 → SSH as `charix`  |
| User       | `charix` (password `Charix!2#4%6&8(0`)                                           |
| Privesc    | root-owned VNC on localhost:5901 → SSH port-forward → `vncviewer` with `secret`  |
| Flags      | user.txt as charix; root.txt via VNC desktop as root                             |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.1.254
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-10 15:58 +0100
Warning: 10.129.1.254 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.129.1.254
Host is up (0.047s latency).
Not shown: 40568 filtered tcp ports (no-response), 24965 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

<img width="1075" height="324" alt="image" src="https://github.com/user-attachments/assets/a868b853-163f-4405-9659-41339ec62201" />

```
nmap -sCV -p22,80 10.129.1.254
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-10 16:01 +0100
Nmap scan report for 10.129.1.254
Host is up (0.018s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
|_http-server-header: Apache/2.4.29 (FreeBSD) PHP/5.6.32
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OS: FreeBSD; CPE: cpe:/o:freebsd:freebsd

Nmap done: 1 IP address (1 host up) scanned in 8.62 seconds
```

<img width="1201" height="490" alt="image" src="https://github.com/user-attachments/assets/5dccb38c-47a8-44f6-8e85-337b7c94bdac" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 7.2 on FreeBSD. Open, but no credentials yet, so a target to return to once I find some.
- **Port 80 (HTTP)** — Apache/PHP web application. This is the entry point, so I'll focus here first.

---
## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH into the server given valid creds
WHAT I AM GOING TO TRY: Set it aside
WHY: No credentials yet
RESULT: N/A
WHAT NEXT:
- Investigate the web application
- Dir-fuzz for hidden directories
- Identify the tech stack in use
```

```
OBSERVATION:
- The web app is a "script testing" site that runs named PHP files; browse.php takes a
  file parameter (browse.php?file=<name>)
- Submitting listfiles.php lists files in the directory, and pwdbackup.txt stood out (it
  wasn't linked in the page, but showed up in the listing) likely a password backup
- Reading browse.php?file=pwdbackup.txt returned a blob the file itself says is "encoded
  at least 13 times"
- In CyberChef: stripped whitespace, ran Magic → base64; decoding base64 13 times produced
  a plausible password (Charix!2#4%6&8(0)
- Since browse.php?file= loads arbitrary files, I tested LFI with /etc/passwd and got a valid
  response. Only root and charix have real login shells; the recovered password contains
  the string "charix", so it almost certainly belongs to that user
 - Nothing interesting was found in dir fuzz, tech stack.
WHAT IT TELLS ME:
- The file parameter is a local file include, and the decoded password likely belongs to
  charix → try it over SSH
WHAT I AM GOING TO TRY:
- SSH in as charix with the decoded password
WHY:
- The credential's content ties it to charix, so reuse over SSH is very likely
RESULT:
- SSH as charix succeeded; read user.txt
WHAT NEXT:
- Enumerate the system for a privesc path
```

<img width="892" height="421" alt="image" src="https://github.com/user-attachments/assets/cfe2422d-c7dd-4e3d-be0b-306ba914bdf4" />

<img width="1192" height="218" alt="image" src="https://github.com/user-attachments/assets/b6c08030-4929-4949-ad10-a97dedb6425e" />

<img width="2420" height="301" alt="image" src="https://github.com/user-attachments/assets/3e958736-9d25-4d41-81aa-a074f24bcd34" />

<img width="2755" height="1216" alt="image" src="https://github.com/user-attachments/assets/11580dcb-a604-422d-b113-44f3bf8f07cd" />

<img width="1007" height="635" alt="image" src="https://github.com/user-attachments/assets/14152430-4068-481a-8636-a9e982c950e8" />

<img width="1005" height="861" alt="image" src="https://github.com/user-attachments/assets/15959627-ae78-4fca-a67f-5368f2521201" />

```
OBSERVATION:
- charix's home held secrets.zip; I pulled it back to my attack box
- Unzipping prompted for a password; reusing charix's SSH password worked and extracted a
  file named "secret"
- `file secret` reports "Non-ISO extended-ASCII text, with no line terminators" and it's
  ~8 bytes of binary-looking data not a text password
- Process/port enumeration on the box shows a VNC server (Xvnc :1) running AS ROOT, bound to
  localhost (127.0.0.1:5901) which is why it never appeared in my external nmap
- VNC supports password-file auth (an obfuscated 8-byte DES secret), which matches the size
  and format of the extracted "secret" file exactly
WHAT IT TELLS ME:
- The "secret" file is the VNC password file for the root-owned VNC server. If I can reach
  the internal 5901 and authenticate with it, I get a root desktop session
WHAT I AM GOING TO TRY:
- SSH local port-forward 5901 from the box to my attack machine, then connect with
  vncviewer using the extracted "secret" file
WHY:
- 5901 is bound to localhost only, so I need charix's SSH access to tunnel to it; the VNC
  service authenticates with the password file, not a typed password
RESULT:
- vncviewer over the tunnel opened a root-owned desktop; read root.txt
```

<img width="929" height="113" alt="image" src="https://github.com/user-attachments/assets/9a6e7e6a-79b6-43f2-9ca9-23550b38194f" />

<img width="969" height="1083" alt="image" src="https://github.com/user-attachments/assets/0d3244aa-823a-4555-b03e-f083446718ed" />

<img width="1002" height="333" alt="image" src="https://github.com/user-attachments/assets/0bd38f1d-4f2a-42d7-a1d6-076648f1f759" />

<img width="1067" height="769" alt="image" src="https://github.com/user-attachments/assets/669595ba-6f49-49ac-ae04-aa7aee198bca" />

<img width="1403" height="729" alt="image" src="https://github.com/user-attachments/assets/3bcf74c0-d6aa-420d-823f-b92a1f8709a5" />

---

## Foothold

**How I got in:** The web app's `browse.php?file=` parameter loaded arbitrary local files, a Local File Inclusion. Listing the directory via `listfiles.php` exposed `pwdbackup.txt`, whose contents were base64 encoded ~13 times; decoding it thirteen times yielded `Charix!2#4%6&8(0`. Reading `/etc/passwd` through the same LFI showed only `root` and `charix` with login shells, and since the password string contained "charix", I reused it over SSH and logged in as **charix**, reading user.txt.

**Command / exploit used:**

```
# LFI: list files, then read the backup
curl "http://10.129.1.254/browse.php?file=listfiles.php"
curl "http://10.129.1.254/browse.php?file=pwdbackup.txt"

# decode base64 13 times (CyberChef Magic, or this one-liner)
data=$(cat pwdbackup.txt); for i in $(seq 1 13); do data=$(echo "$data" | base64 -d); done; echo "$data"
# → Charix!2#4%6&8(0

# confirm users via the same LFI
curl "http://10.129.1.254/browse.php?file=/etc/passwd"

# reuse the password over SSH
ssh charix@10.129.1.254        # → user.txt
```

**Why it worked:** `browse.php` passed user input straight into a file read with no path restriction, so any local file (including `/etc/passwd`) was disclosable. The application author left a lightly "protected" password backup readable through that same include but repeated base64 encoding is encoding, not encryption, so it was trivially reversible. Password reuse between that backup and charix's SSH account turned file disclosure into an interactive shell.

---

## Privilege Escalation

**Path taken (charix → root):** charix's home contained `secrets.zip`, which unzipped (reusing charix's password) to a small binary file named `secret`. Enumeration showed a **VNC server (`Xvnc :1`) running as root but bound to `127.0.0.1:5901`**, so it was invisible externally. The `secret` file is a **VNC password file** (an obfuscated ~8-byte DES secret), matching its size and format. I set up an **SSH local port-forward** to reach the localhost-only VNC port, then connected with `vncviewer` using `secret` landing a root-owned desktop and reading root.txt.

**Command / exploit used:**

```
# forward the localhost-only VNC port to my box
ssh -L 5901:127.0.0.1:5901 charix@10.129.1.254

# in another terminal, authenticate with the extracted VNC password file
vncviewer -passwd secret 127.0.0.1:5901        # → root desktop → root.txt
```

**Why it worked:** The VNC service ran with root privileges and was "protected" only by being bound to localhost, a network-level restriction, not real authentication. Once charix's SSH access let me tunnel into that local port, the only remaining gate was the VNC password, and that secret had been left sitting in charix's home inside a zip encrypted with the same password charix used everywhere. VNC's password-file auth accepts the obfuscated secret directly, so no cracking was needed to reach the port, present the file, get a root session.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Used an LFI in `browse.php?file=` to read a base64-×13 password backup and `/etc/passwd`, then reused the decoded credential to SSH in as charix.
2. **Privesc technique in one sentence:** SSH-tunnelled to a root-owned VNC server bound to localhost and authenticated with a VNC password file recovered from charix's home, opening a root desktop.
