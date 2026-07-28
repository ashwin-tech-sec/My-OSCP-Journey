# Devvortex — Easy — Linux

**Date:** 27 July 2026

---

## Tags
`#linux` `#web` `#vhost-fuzzing` `#joomla` `#cve-2023-23752` `#information-disclosure` `#template-rce` `#mysql` `#hash-cracking` `#apport-cli` `#cve-2023-1326` `#pager-escape` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | devvortex.htb / dev.devvortex.htb (10.129.229.146)             |
| OS         | Linux (Ubuntu)                                                 |
| Open ports | 22 (SSH), 80 (HTTP)                                            |
| Services   | OpenSSH 8.2p1; nginx 1.18.0                                    |
| Web app    | Joomla 4.2.6 (on the dev subdomain)                           |
| Foothold   | vhost fuzz → Joomla CVE-2023-23752 info disclosure → admin → template RCE |
| Users      | www-data → logan → root                                        |
| Privesc    | DB creds → crack logan's hash; sudo apport-cli pager escape (CVE-2023-1326) → root |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.229.146

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-27 22:58 +0100
Nmap scan report for 10.129.229.146
Host is up (0.021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 20.85 seconds
```

<img width="858" height="310" alt="image" src="https://github.com/user-attachments/assets/45b6e6e8-9c0c-4767-80e6-e8f6b94557de" />

```
nmap -sC -sV -p22,80 10.129.229.146

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-27 22:59 +0100
Nmap scan report for 10.129.229.146
Host is up (0.018s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devvortex.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.25 seconds
```

<img width="1211" height="480" alt="image" src="https://github.com/user-attachments/assets/a22a25c1-9443-4a86-a99f-09aa01a15fb4" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.2p1 on Ubuntu. Open, but no credentials yet, a target to return to once I find some.
- **Port 80 (HTTP)** — nginx 1.18.0, redirecting to `devvortex.htb`. My entry point; the vhost redirect means I add `devvortex.htb` to `/etc/hosts` and should also fuzz for other vhosts.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Port 80 redirects to devvortex.htb, add it to /etc/hosts and investigate
- Directory-fuzz for hidden paths
- vhost-fuzz for additional subdomains
- Identify the tech stack
```

```
OBSERVATION:
- The main site is a "dynamic web development" company page with little functionality
- No interesting directories found
- vhost fuzzing reveals a "dev" subdomain: dev.devvortex.htb
- Nothing notable in the stack
WHAT IT TELLS ME: I need to investigate dev.devvortex.htb, dev subdomains are often less hardened
WHAT I AM GOING TO TRY:
- Add dev.devvortex.htb to /etc/hosts and browse it
- Directory-fuzz the subdomain
- Identify its tech stack
WHY: A "dev" host can be a treasure trove of exposed functionality
RESULT:
- The subdomain looks similar to the main site with little functionality
- Wappalyzer shows nothing immediate, but /robots.txt indicates Joomla is installed
WHAT NEXT:
- Investigate Joomla; knowing the exact version would let me target a specific exploit
```

<img width="2556" height="952" alt="image" src="https://github.com/user-attachments/assets/2c91a00a-cde1-4ab1-8719-d1f9e0a94e59" />

<img width="2025" height="696" alt="image" src="https://github.com/user-attachments/assets/c7df225b-12b8-4048-b03a-9245af877017" />

<img width="513" height="571" alt="image" src="https://github.com/user-attachments/assets/30ee3ba7-0aa2-45d7-bf05-034594dd4ca0" />

<img width="2560" height="958" alt="image" src="https://github.com/user-attachments/assets/eaed7b40-cb4a-4e9a-83dd-1ee169772adc" />

<img width="985" height="356" alt="image" src="https://github.com/user-attachments/assets/dbe6c544-f564-48b5-bf7a-7a820cdd7983" />

```
OBSERVATION:
- Joomla's version is disclosed at /administrator/manifests/files/joomla.xml, here it's 4.2.6
- Joomla 4.2.6 has several linked CVEs; the useful one is CVE-2023-23752, an information-disclosure
  flaw letting unauthenticated users read sensitive configuration (including DB credentials)
- There's a Joomla admin panel at /administrator
WHAT IT TELLS ME: I can abuse CVE-2023-23752 to read sensitive config, potentially credentials
WHAT I AM GOING TO TRY: Exploit CVE-2023-23752 to dump the config
WHY: Exposed config often contains reusable credentials
RESULT:
- Used the PoC (https://github.com/Acceis/exploit-CVE-2023-23752) and recovered a set of credentials
- They didn't work for SSH, but DID log me into the Joomla admin panel
WHAT NEXT:
- With admin access to Joomla, work toward code execution (reverse shell)
```

<img width="1042" height="753" alt="image" src="https://github.com/user-attachments/assets/248df007-b0a4-49e5-a534-eba129e1f040" />

<img width="939" height="262" alt="image" src="https://github.com/user-attachments/assets/2ba8fab6-ef52-4d4a-87be-22112db74779" />

<img width="2374" height="1068" alt="image" src="https://github.com/user-attachments/assets/0309f81a-77f4-4f4d-9ead-0c04cd8cff65" />

<img width="1050" height="1074" alt="image" src="https://github.com/user-attachments/assets/1c65b212-e69a-489e-92bd-d54b9574a87a" />

<img width="2558" height="1086" alt="image" src="https://github.com/user-attachments/assets/d62cbaad-a057-4b37-9265-b186114ac293" />

```
OBSERVATION: In the admin panel: System > Site Templates > Cassiopeia > error.php is an editable PHP file
WHAT IT TELLS ME: If I can edit error.php, I can plant PHP and trigger it for a reverse shell
WHAT I AM GOING TO TRY: Overwrite error.php with a PHP reverse shell, then request a non-existent
  page (or the template file directly) to execute it
WHY: A writable server-side PHP template = arbitrary code execution as the web user
RESULT: Triggered the modified template and got a shell as www-data
WHAT NEXT: Enumerate the system
```

<img width="1655" height="916" alt="image" src="https://github.com/user-attachments/assets/afc7731b-ff94-4c5a-8190-493a043fb9f7" />

<img width="1532" height="202" alt="image" src="https://github.com/user-attachments/assets/256a7d2c-420d-45d3-b453-329c8880e874" />

<img width="1200" height="292" alt="image" src="https://github.com/user-attachments/assets/73ae81e7-6fc8-4404-b604-c1b5f7481a55" />

```
OBSERVATION:
- /home has a single user, logan (user.txt is there but not yet readable as www-data)
- Port 3306 (MySQL) is listening locally and the creds I recovered earlier were DB credentials
WHAT IT TELLS ME: I can reuse the recovered DB creds to connect to MySQL locally
WHAT I AM GOING TO TRY: Connect to the database and look for user credentials
WHY: The Joomla user table stores password hashes worth cracking
RESULT:
- Connected to MySQL and dumped logan's password hash from the Joomla users table
- Cracked it with john, and the password worked for SSH. logged in as logan and read user.txt
WHAT NEXT: Enumerate as logan for a privesc path
```

<img width="876" height="258" alt="image" src="https://github.com/user-attachments/assets/9dd58ce7-379a-4f6a-9bd2-fe19bfb4af8a" />

<img width="1552" height="621" alt="image" src="https://github.com/user-attachments/assets/08fa8919-ac0b-4816-8abb-415802bf6424" />

<img width="1050" height="978" alt="image" src="https://github.com/user-attachments/assets/aabc9da0-c9a0-454a-964a-76ac612a459e" />

<img width="1581" height="772" alt="image" src="https://github.com/user-attachments/assets/3ba1863c-b94a-4ce8-a6ec-84f3464f0ca1" />

<img width="1296" height="486" alt="image" src="https://github.com/user-attachments/assets/9281aefd-399d-41ec-88ba-3c233a70bf9d" />

<img width="1062" height="1054" alt="image" src="https://github.com/user-attachments/assets/0c5f45bf-4e33-46db-b490-502e43c7be32" />

```
OBSERVATION: sudo -l shows logan can run /usr/bin/apport-cli as root without a password
WHAT IT TELLS ME: apport-cli is a candidate for a GTFOBins-style escape, check how to abuse it
WHAT I AM GOING TO TRY: Look up apport-cli on GTFOBins and use the escape
WHY: A sudo-runnable binary with a shell/pager escape is a common root vector
RESULT:
- apport-cli (run with sudo -f to file a crash report) pipes its report through a pager (less).
  This is CVE-2023-1326 from the less pager I ran !/bin/bash to spawn a shell, which runs as root
- Got a root shell and read root.txt
```

<img width="1506" height="173" alt="image" src="https://github.com/user-attachments/assets/26437278-d353-4a65-ad99-040ccd645035" />

<img width="1225" height="656" alt="image" src="https://github.com/user-attachments/assets/bfd9066f-0574-4000-a0b4-2c81754f0fd8" />

<img width="1123" height="366" alt="image" src="https://github.com/user-attachments/assets/44271792-6141-4e34-be1e-0ebff2299317" />

<img width="850" height="474" alt="image" src="https://github.com/user-attachments/assets/2d7b62f8-1714-4fbf-8c63-5574a45781e6" />

<img width="1095" height="886" alt="image" src="https://github.com/user-attachments/assets/dad3af52-9304-4132-b21d-58ddfaf75de5" />

<img width="980" height="334" alt="image" src="https://github.com/user-attachments/assets/0c29bdc7-4154-4bdc-a66e-fef14d931f29" />

---

## Foothold

**How I got in:** vhost fuzzing revealed `dev.devvortex.htb`, running **Joomla 4.2.6** (version leaked at `/administrator/manifests/files/joomla.xml`). That version is vulnerable to **CVE-2023-23752**, an unauthenticated information-disclosure flaw that exposes the config API, including plaintext database credentials and the Joomla admin login. Those creds didn't work for SSH but did log into the Joomla admin panel, where I edited the **Cassiopeia** template's `error.php` to include a PHP reverse shell and triggered it for a shell as **www-data**.

**Command / exploit used:**
```
echo "10.129.229.146  devvortex.htb dev.devvortex.htb" | sudo tee -a /etc/hosts

# confirm version
curl -s http://dev.devvortex.htb/administrator/manifests/files/joomla.xml | grep version

# CVE-2023-23752 — leak users + DB creds
ruby exploit.rb http://dev.devvortex.htb     # Acceis PoC → username + DB password

# log into /administrator, then:
# System > Site Templates > Cassiopeia > error.php → paste PHP reverse shell → save → trigger
nc -lnvp <LPORT>          # → shell as www-data
```

**Why it worked:** CVE-2023-23752 is a broken access-control bug, the Joomla REST API endpoints (`/api/index.php/v1/config/...`) failed to enforce authentication, so anyone could read the site config and its stored credentials. With admin access, Joomla legitimately lets an administrator edit template PHP files, which is effectively a built-in code-execution feature once you're in.

---

## Privilege Escalation

**Path taken (www-data → logan → root):**

1. **Lateral move via the database.** MySQL was listening locally, and the DB credentials leaked by CVE-2023-23752 worked. I dumped logan's password hash from the Joomla users table, cracked it with john, and the password was reused for logan's system account, SSH in and read user.txt.
2. **Root via apport-cli pager escape.** `sudo -l` showed logan could run `/usr/bin/apport-cli` as root with no password. Running `sudo apport-cli -f` to file a crash report eventually displays the report through the **less** pager. This is **CVE-2023-1326**: from inside less I ran `!/bin/bash`, which spawns a shell and since apport-cli was running as root via sudo, that shell is a **root** shell. Read root.txt.

```
# as logan:
sudo /usr/bin/apport-cli -f        # choose a crash type, proceed to "View report"
# at the less pager prompt:
!/bin/bash                          # → root shell
```

**Why it worked:** Two issues chained. First, password reuse the Joomla DB hash, once cracked, was also logan's login. Second, and the root cause: giving a user passwordless sudo on `apport-cli` is dangerous because apport pipes output through a pager, and pagers like `less` can shell out (`!cmd`). Running the pager as root means that shell-out runs as root, a classic "GTFOBins" interactive-program escape (CVE-2023-1326).

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Found a dev vhost running Joomla 4.2.6, used CVE-2023-23752 to leak admin + DB credentials, logged into the admin panel, and edited the Cassiopeia template's `error.php` into a PHP reverse shell for a www-data shell.
2. **Privesc technique in one sentence:** Reused the leaked DB credentials to crack logan's hash from MySQL for SSH, then abused passwordless sudo on apport-cli (CVE-2023-1326 pager escape `!/bin/bash` from less) to become root.
