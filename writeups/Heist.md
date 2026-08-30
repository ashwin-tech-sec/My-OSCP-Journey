# Heist — Easy — Windows

**Date:** 24 - August - 2026

---

## Tags

`#windows` `#web` `#cisco-config` `#cisco-type7` `#cisco-type5` `#hash-cracking` `#smb` `#rpcclient` `#rid-brute` `#password-spraying` `#winrm` `#firefox` `#procdump` `#memory-dump` `#credential-reuse` `#privesc`

---

## What I Know

| Item       | Detail                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------ |
| Target     | 10.129.96.157 (host: Heist, SUPPORTDESK)                                                    |
| OS         | Windows (IIS 10.0)                                                                          |
| Open ports | 80, 135, 445, 5985, 49669                                                                   |
| Services   | IIS 10.0 (80); MSRPC (135, 49669); SMB (445); WinRM (5985)                                  |
| Foothold   | Guest login -> Cisco config -> crack Type7/Type5 -> hazard SMB -> RID-brute -> spray -> chase |
| User       | `chase` via WinRM (found by RID-brute + password spray) -> user.txt                         |
| Privesc    | Firefox process memory dump (procdump) -> admin site creds reused as Administrator          |
| Flags      | user.txt as chase; root.txt as Administrator                                                |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.96.157
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
49669/tcp open  unknown
```
<img width="858" height="395" alt="image" src="https://github.com/user-attachments/assets/f3d2de3f-d551-45b6-b64c-32856e397c38" />

```
nmap -sC -sV -p80,135,445,5985 10.129.96.157
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-title: Support Login Page
|_Requested resource was login.php
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
<img width="1199" height="874" alt="image" src="https://github.com/user-attachments/assets/1ab1d9dd-7819-4b9a-b940-8b1c05bc6578" />

**What each port tells me:**
- **Port 80 (HTTP)** — IIS serving a "Support Login Page" (PHP). The main entry point, focus here first.
- **Port 135 (MSRPC)** — Standard Windows RPC endpoint mapper; confirms a typical Windows host.
- **Port 445 (SMB)** — Worth checking for anonymous access or reusable creds; later the RID-brute surface.
- **Port 5985 (WinRM)** — Remote management / PowerShell Remoting; the shell route once I have creds.
- **Port 49669 (MSRPC)** — Dynamic high-numbered RPC port; not directly actionable without specific enumeration.

---

## Enumeration Log

```
OBSERVATION:
- The immediate targets from the scan are 80 and 445
- SMB does not allow anonymous login
- Port 80 is the entry point to a foothold
- The web app is a login page; admin:admin fails, and the username field expects an email
- There's a "login as guest" option; as guest I see a support conversation between user
  "hazard" and an admin, with an attached Cisco router config file
- The config discloses credentials (Cisco password types)
WHAT IT TELLS ME:
- Recover the credentials from the Cisco config
WHAT I AM GOING TO TRY:
- Decrypt/crack the config passwords
WHY:
- Recovered creds can be reused against SMB/WinRM on this host
RESULT:
- The config held three secrets: one Cisco TYPE 7 (reversible weak "encryption", decrypt
  instantly) and Cisco TYPE 5 (salted MD5, crack with john/hashcat -m 500). Recovered all
  three plaintext passwords, plus the username "hazard" from the ticket
- Tried the creds directly but the web/login didn't advance, so I pivoted to SMB/RPC
- hazard : stealth1agent is a valid SMB credential. rpcclient with it can't enumdomusers, but
  I RID-BRUTE (lookupsid / --rid-brute) to enumerate usernames -> got several accounts
  (Chase, Jason, etc.)
- I then PASSWORD-SPRAYED the known config passwords against those usernames and found
  chase : Q4)sJu\Y8qz*A3?d
- chase's creds work over WinRM: evil-winrm as chase -> read user.txt
WHAT NEXT:
- Enumerate as chase for a privesc path
```

<img width="616" height="110" alt="image" src="https://github.com/user-attachments/assets/60a3c594-103c-421f-becc-21647e1dc144" />

> SMB share not accessible anonymously

<img width="2556" height="850" alt="image" src="https://github.com/user-attachments/assets/a60a7c38-0b1a-4d62-9c8c-3edd19d436bd" />

> The index page which is a login page with login-as-guest functionality

<img width="2556" height="795" alt="image" src="https://github.com/user-attachments/assets/81432d1d-7b71-4f5c-a119-b45df195eb1c" />

> The conversation between user hazard and support admin about a cisco router issue with config attachment

<img width="648" height="793" alt="image" src="https://github.com/user-attachments/assets/c6a38e11-b6f7-4413-9b94-e4ab1feac8c3" />

> The config file containing the password secrets: one Cisco Type 7 (decrypt) and Cisco Type 5 (crack)

<img width="903" height="170" alt="image" src="https://github.com/user-attachments/assets/4a985a23-420f-47a0-bb41-1b34ca3e77e4" />

> password decrypting using cisco type 7

<img width="903" height="170" alt="image" src="https://github.com/user-attachments/assets/0d92da67-e60b-42d7-b427-d872932a082b" />

> password decrypting using cisco type 7

<img width="1249" height="441" alt="image" src="https://github.com/user-attachments/assets/76fe536c-7c42-48a4-adf2-c8915717a80b" />

> cracking the Cisco Type 5 (salted MD5) secret with john

<img width="1634" height="300" alt="image" src="https://github.com/user-attachments/assets/42dac107-4ae0-4309-ac48-5d6051efdbc7" />

> Tried the decrypted creds directly for hazard without success on the web login

<img width="1003" height="444" alt="image" src="https://github.com/user-attachments/assets/d4b565cb-eaa6-4ec8-9e19-4a56aa7691be" />

> Since 445 was open, I used hazard's creds with rpcclient and RID-brute to enumerate usernames

<img width="2170" height="1058" alt="image" src="https://github.com/user-attachments/assets/a26abdaa-7e41-4e8d-a34c-0adfdf0418c2" />

> Password-spraying the known passwords across the discovered usernames found Chase:Q4)sJu\Y8qz*A3?d

<img width="1654" height="676" alt="image" src="https://github.com/user-attachments/assets/e76a29d4-cf6f-41e4-a4e2-74de644f8433" />

> I logged into the server over WinRM with chase's creds and read user.txt

```
OBSERVATION:
- Enumerating as chase, I found todo.txt (3 items, one already done) referencing checking the
  issues list, and winPEAS repeatedly surfaced Mozilla Firefox running
WHAT IT TELLS ME:
- If Firefox has an active session where creds were entered (the admin checking the support
  issues list), a memory dump of the process may still contain those plaintext creds
WHAT I AM GOING TO TRY:
- Find the Firefox PID(s), dump process memory with procdump, and search the dump for creds
WHY:
- Browsers keep submitted form values in memory; dumping and stringing the process can recover
  a plaintext login, which I can then try to reuse
RESULT:
- Multiple firefox.exe processes were running; I used procdump (Sysinternals) to dump one
- Moved the dump to my attack box and ran strings | grep login. Because the file was huge, the
  transfer was slow, but running strings before it finished still produced a hit:
  email/username = admin@support.htb, password = 4dD!5}x/re8]FBuZ
- That is the support-site admin's password, and it is REUSED as the Windows Administrator
  password. evil-winrm as Administrator -> read root.txt
```

<img width="713" height="194" alt="image" src="https://github.com/user-attachments/assets/c9e7e452-8fba-4807-8c15-be5adc7e18a5" />

> the todo.txt file showing 3 items, of which some need fixing and one is done

<img width="1969" height="628" alt="image" src="https://github.com/user-attachments/assets/fca79670-fbd1-43cb-8bcf-1c0162db0f2b" />

> Shows the presence of Mozilla Firefox

<img width="1567" height="125" alt="image" src="https://github.com/user-attachments/assets/65e954b5-412b-4c6c-9f07-1580df2a1695" />

> Shows the presence of the Mozilla Firefox db

<img width="1024" height="575" alt="image" src="https://github.com/user-attachments/assets/76cad42d-9d6a-455d-bf0f-726ee7c54580" />

> Finding the process id of Firefox

<img width="1426" height="1109" alt="image" src="https://github.com/user-attachments/assets/69067f2a-f130-4947-94b6-eb22c1cd1a36" />

> Generating a memory dump with procdump and transferring it to my attack machine

<img width="1715" height="406" alt="image" src="https://github.com/user-attachments/assets/804d1454-25a8-4c45-bc39-4b6fccc0127a" />

> running strings on the dump and filtering for "login" to find leaked creds

<img width="1622" height="643" alt="image" src="https://github.com/user-attachments/assets/ff505a3e-9ea3-4178-9873-d4ccd77e84d7" />

> using the recovered creds to gain access as Administrator and read root.txt

---

## Foothold

**How I got in:** The support site allowed **guest login**, which exposed a ticket containing an attached **Cisco IOS config**. The config held a **Cisco Type 7** secret (reversible, decrypt instantly) and **Cisco Type 5** hashes (salted MD5, cracked with john), yielding three plaintext passwords plus the username `hazard`. `hazard:stealth1agent` was a valid **SMB** credential; since `enumdomusers` was blocked, I used **RID brute-forcing** via rpcclient to enumerate usernames, then **password-sprayed** the known passwords across them to find `chase:Q4)sJu\Y8qz*A3?d`. chase logs in over **WinRM** (evil-winrm) for user.txt.

**Command / exploit used:**

```
# guest login -> download the Cisco config from the ticket
# decrypt Type 7 (instant) and crack Type 5
# (Type 7: any cisco type7 decoder; Type 5: john/hashcat -m 500)
john --format=md5crypt --wordlist=rockyou.txt type5.hash

# hazard's creds are valid on SMB; RID-brute usernames, then spray
rpcclient -U 'hazard%stealth1agent' 10.129.96.157      # lookupnames/lookupsids
crackmapexec smb 10.129.96.157 -u hazard -p 'stealth1agent' --rid-brute
crackmapexec smb 10.129.96.157 -u users.txt -p passwords.txt   # spray -> chase valid

# shell as chase over WinRM
evil-winrm -i 10.129.96.157 -u chase -p 'Q4)sJu\Y8qz*A3?d'      # -> user.txt
```

**Why it worked:** A public-facing "support" portal exposed a real Cisco config through guest access, and Cisco's legacy password formats are weak: Type 7 is trivially reversible and Type 5 (unsalted-ish MD5crypt) cracks against rockyou. Those recovered passwords, combined with usernames harvested by walking RIDs over RPC (which works even when direct user enumeration is blocked), let a simple spray land a valid WinRM account. The whole foothold is credential recovery and reuse, no memory-corruption exploit.

---

## Privilege Escalation

**Path taken (chase -> Administrator):** As chase, `todo.txt` referenced regularly checking the issues list, and winPEAS showed **Firefox** running. Since the admin checks the support site in that browser, its process memory likely still held the submitted login. I found the `firefox.exe` PIDs and used **procdump** (Sysinternals) to dump the process, moved the dump to my box, and ran `strings | grep login`, recovering `admin@support.htb : 4dD!5}x/re8]FBuZ`. That support-site admin password is **reused as the Windows Administrator password**, so evil-winrm as Administrator gave root.txt.

**Command / exploit used:**

```
# on the target as chase
Get-Process firefox                                 # find PIDs
.\procdump.exe -accepteula -ma <firefox_pid> fx.dmp # dump process memory
# exfil fx.dmp, then on the attack box:
strings fx.dmp | grep -i "login\|password"          # -> admin@support.htb : 4dD!5}x/re8]FBuZ

# password is reused for the Administrator account
evil-winrm -i 10.129.96.157 -u Administrator -p '4dD!5}x/re8]FBuZ'   # -> root.txt
```

**Why it worked:** A privileged user was actively using Firefox to log into the support portal, and web browsers keep submitted form data (including passwords) in process memory. Dumping that memory with a legitimate Microsoft tool and stringing it recovered the plaintext credential, no cracking required. The escalation is pure credential reuse: the same password protected both the web admin account and the local Windows Administrator, so recovering one handed over the other.

---

### Reflection
1. **Foothold technique in one sentence:** Grabbed a Cisco config from a guest-accessible support ticket, cracked its Type 7/Type 5 secrets, used hazard's SMB creds to RID-brute usernames, then sprayed the known passwords to find chase and logged in over WinRM.
2. **Privesc technique in one sentence:** Dumped a running Firefox process with procdump and stringed the memory to recover the support-admin password, which was reused as the Windows Administrator password.
