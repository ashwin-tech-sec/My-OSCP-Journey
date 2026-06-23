# Codify — Easy — Linux

**Date:** 23 June 2026

---

## Tags
`#linux` `#web` `#nodejs` `#vm2-sandbox-escape` `#sqlite` `#hash-cracking` `#bash-pattern-match` `#pspy` `#password-reuse` `#privesc`

---

## What I Know

| Item       | Detail                                                       |
| ---------- | ----------------------------------------------------------- |
| Target     | codify.htb (10.129.20.40)                                   |
| OS         | Linux (Ubuntu)                                              |
| Open ports | 22 (SSH), 80 (HTTP), 3000 (HTTP)                            |
| Services   | OpenSSH 8.9p1; Apache 2.4.52 (80); Node.js/Express (3000)   |
| Web app    | Node.js code sandbox using vm2 3.9.16                       |
| Users      | svc → joshua → root                                         |
| Foothold   | vm2 3.9.16 sandbox escape → RCE as svc                      |
| Privesc    | tickets.db hash → joshua; sudo bash script pattern-match bug → root password |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.20.40

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-23 13:04 +0100
Nmap scan report for 10.129.20.40
Host is up (0.051s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3000/tcp open  ppp

Nmap done: 1 IP address (1 host up) scanned in 20.48 seconds
```

<img width="933" height="338" alt="image" src="https://github.com/user-attachments/assets/3ebd04da-13d5-48c5-8353-ce5f71c1cd23" />

```
nmap -sC -sV -p22,80,3000 10.129.20.40

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-23 17:29 +0100
Nmap scan report for codify.htb (10.129.20.40)
Host is up (0.020s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 96:07:1c:c6:77:3e:07:a0:cc:6f:24:19:74:4d:57:0b (ECDSA)
|_  256 0b:a4:c0:cf:e2:3b:95:ae:f6:f5:df:7d:0c:88:d6:ce (ED25519)
80/tcp   open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
3000/tcp open  http    Node.js Express framework
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 73.74 seconds
```
<img width="1268" height="474" alt="image" src="https://github.com/user-attachments/assets/73f9a716-4136-403e-bca5-18a9c658fb96" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but I have no credentials yet. A target to return to once I find some.
- **Port 80 (HTTP)** — Apache 2.4.52, redirecting to codify.htb. The main web front end.
- **Port 3000 (HTTP)** — Node.js/Express. The same application appears to be served here too; a second view of the web app and a strong hint the foothold is Node-related.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Port 80 redirects to codify.htb, add it to /etc/hosts and investigate
- Directory-fuzz for hidden paths
- vhost-fuzz for additional subdomains
- Identify the application's tech stack
```

```
OBSERVATION:
- The app is a sandbox that lets users run Node.js code. Browsing the pages, two things stand out:
  1. It has security controls that block direct abuse of Node.js for a straightforward reverse shell
  2. It uses the vm2 3.9.16 library for sandboxing (linked from /about)
- No hidden directories or vhosts found
- Wappalyzer shows nothing else notable in the stack
WHAT IT TELLS ME:
- Direct module abuse is restricted, so a naive shell won't work
- Knowing the exact sandbox library + version is the real lead, sandbox libraries have a history of escape CVEs
WHAT I AM GOING TO TRY: Research known vulnerabilities in vm2 3.9.16
WHY: To find a sandbox-escape that turns "run code in a sandbox" into real RCE
RESULT:
- Used the vm2 sandbox-escape PoC from
  https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244
- Adapted it with a reverse-shell command (the first one-liner I tried didn't fire; after a few attempts with different
  payloads, one worked) and caught a shell as svc
WHAT NEXT:
- Read user.txt
- Enumerate the system
```

<img width="2090" height="900" alt="image" src="https://github.com/user-attachments/assets/aff1013c-bc39-4daf-ac3a-d498441ad169" />

<img width="1728" height="1010" alt="image" src="https://github.com/user-attachments/assets/e117b0ec-e230-40fe-96ff-0a794aa79d4d" />

<img width="1882" height="809" alt="image" src="https://github.com/user-attachments/assets/16b552f8-0536-4072-89bc-867199edf273" />

<img width="1390" height="242" alt="image" src="https://github.com/user-attachments/assets/a307ffc1-ab86-4a5b-8731-2f1e9e04d324" />

<img width="1822" height="949" alt="image" src="https://github.com/user-attachments/assets/7e52fcff-8a32-45e2-a60c-8acda26dde41" />

<img width="798" height="177" alt="image" src="https://github.com/user-attachments/assets/c2d7fbda-26f5-4c59-9db7-b257a95f3ed9" />

```
OBSERVATION:
- user.txt is NOT in svc's home; there is another user, joshua
- Under /var/www/ there are three folders: editor, contact, html
  - html: default Apache welcome page
  - editor: the app exposed on port 80
  - contact: more interesting. its index.js contains a hardcoded/static session token, and there's a tickets.db file
WHAT IT TELLS ME:
- I need to move laterally to joshua to get user.txt
- The static session token in contact could let me forge a valid session for that app
- tickets.db (SQLite) may hold useful data
WHAT I AM GOING TO TRY:
- Find joshua's password to move laterally
- Read tickets.db and inspect its contents
WHY:
- svc lacks user.txt, so joshua likely holds it
- A local DB file is a classic spot for stored credentials/hashes
RESULT: Extracted a password hash from tickets.db
WHAT NEXT:
- Crack the hash and log in as joshua
- Enumerate as joshua
```

<img width="616" height="109" alt="image" src="https://github.com/user-attachments/assets/7d2f9d2a-da60-4e13-89bc-645fa09f2685" />

<img width="250" height="32" alt="image" src="https://github.com/user-attachments/assets/b5287acb-d13d-4102-93c7-ff902edec403" /></br>

<img width="128" height="108" alt="image" src="https://github.com/user-attachments/assets/f6fbc126-5227-41b7-950c-40e4f3f14e84" /></br>

<img width="329" height="206" alt="image" src="https://github.com/user-attachments/assets/4c2881e1-125b-461f-b32b-9406327143cd" />

<img width="735" height="106" alt="image" src="https://github.com/user-attachments/assets/0ea9f642-ffa7-4d44-a86a-8d35492df74f" />

<img width="336" height="71" alt="image" src="https://github.com/user-attachments/assets/19971411-ed42-492b-acdc-200df8677ed1" />

<img width="890" height="128" alt="image" src="https://github.com/user-attachments/assets/bd857135-faf8-4513-a167-8a9bae5042d4" />

```
OBSERVATION:
- Cracked joshua's password (from the tickets.db hash), logged in as joshua, and read user.txt
- sudo -l shows joshua can run a custom script as root WITHOUT a password
WHAT IT TELLS ME: If that root-run script is misconfigured, it's my privesc vector
WHAT I AM GOING TO TRY: Read the script and analyse it for a flaw
WHY: A script that runs as root is only as safe as its own logic
RESULT:
- The script backs up a MySQL database and asks for the MySQL root password (DB_PASS)
- It compares the entered password with: if [[ $DB_PASS == $USER_PASS ]]
- Because the right-hand side of == inside [[ ]] is UNQUOTED, bash treats it as a GLOB PATTERN,
  not a literal string so entering "*" matches ANY value and bypasses the check
WHAT NEXT:
- The script uses the real root DB password during the backup process, so I'll run it and snoop the
  process table with pspy64 to capture the actual credentials as they're passed
```

<img width="1215" height="334" alt="image" src="https://github.com/user-attachments/assets/c907bb37-4e6f-40bf-a6b5-6f2aacbb3694" />

<img width="1132" height="274" alt="image" src="https://github.com/user-attachments/assets/f1c998be-0a98-4820-97a4-2130b8327965" />

<img width="361" height="57" alt="image" src="https://github.com/user-attachments/assets/bcf4d296-9299-49c5-895c-3fffbd3cc05e" />

<img width="1648" height="183" alt="image" src="https://github.com/user-attachments/assets/88d3217b-ae3d-4518-a239-2a0651bd9b34" />

<img width="2091" height="755" alt="image" src="https://github.com/user-attachments/assets/17ec5f08-d8be-429d-a8c5-aae6be3b6e75" />

```
OBSERVATION: Ran the script and watched it with pspy64, captured the MySQL/root credentials passed during the backup
WHAT IT TELLS ME: If that password is reused for the system root account, I can become root
WHAT I AM GOING TO TRY: Reuse the captured password to switch to root
WHY: Password reuse between a service/DB account and the system account is common
RESULT: The password worked, su to root and read root.txt
```

<img width="879" height="105" alt="image" src="https://github.com/user-attachments/assets/b9ff9388-140a-4d10-8194-a2fcb3d6040a" />

<img width="2498" height="279" alt="image" src="https://github.com/user-attachments/assets/93a767a9-fe23-4eb0-860a-3f1e0fe28262" />

<img width="512" height="23" alt="image" src="https://github.com/user-attachments/assets/6e2fa554-063d-45b5-810d-e280f796a6a1" />

<img width="1187" height="286" alt="image" src="https://github.com/user-attachments/assets/617be735-d523-49e5-9417-2fa2e6b12a54" />

<img width="1120" height="371" alt="image" src="https://github.com/user-attachments/assets/773fa318-5c56-444a-af59-4ec5653fed27" />

<img width="1795" height="300" alt="image" src="https://github.com/user-attachments/assets/8e8b8320-dc1b-457b-a3d6-e00def859a38" />

<img width="407" height="136" alt="image" src="https://github.com/user-attachments/assets/e930fe10-db3e-4b8d-9b66-2da427cba3f7" />

---

## Foothold

**How I got in:** The web app is a Node.js code sandbox that uses the **vm2 3.9.16** library to isolate user-submitted code. vm2 3.9.16 has a known sandbox-escape vulnerability, code running inside the "sandbox" can break out and execute on the host. Using a public PoC adapted with a reverse-shell payload, I escaped the sandbox and got a shell as **svc**.

**Command / exploit used:**
```
echo "10.129.20.40  codify.htb" | sudo tee -a /etc/hosts
# paste the vm2 escape PoC (https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244)
# into the code editor, with a reverse-shell command as the payload
nc -lnvp <LPORT>          # → shell as svc
```

**Why it worked:** vm2 aims to run untrusted JavaScript safely, but version 3.9.16 has a flaw that lets sandboxed code reach host objects and call out to the OS. Since the app fed my input straight into that vulnerable sandbox, "test some Node.js" became arbitrary code execution on the server.

---

## Privilege Escalation

**Path taken (svc → joshua → root):**

1. **Lateral move via a local SQLite DB.** In `/var/www/contact`, `tickets.db` held a password hash. I cracked it offline, which gave **joshua**'s password. Logged in and read user.txt. (The same folder's `index.js` also exposed a static session token, an alternative way to forge access to that app.)
2. **Root via a vulnerable sudo bash script.** `sudo -l` showed joshua could run a custom DB-backup script as root with no password. The script compares the entered MySQL password using `if [[ $DB_PASS == $USER_PASS ]]` and because the right-hand variable is **unquoted inside `[[ ]]`**, bash evaluates it as a glob pattern rather than a literal string. Supplying `*` matches any stored value, bypassing the password check. I then ran the script and used **pspy64** to snoop the process list, capturing the real root/MySQL password as it was passed during the backup. That password was reused for the system root account, so `su root` succeeded and read root.txt.

```
# bash pattern-match bypass: enter * when the script prompts for the DB password
# meanwhile, in another joshua session:
./pspy64        # watch the process table while the script runs → capture the real password
su root         # reuse the captured password
```

**Why it worked:** Two distinct bugs chained. First, `[[ $a == $b ]]` with an unquoted right side is a classic bash footgun `==` does pattern matching, so an attacker-supplied `*` becomes a wildcard that matches everything, defeating the comparison. Second, the script handled the real root password in a way observable from the process table (pspy), and that password was reused for the root login. Quoting the variable (`"$USER_PASS"`) and not reusing credentials would each have broken the chain.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Escaped the Node.js sandbox by exploiting a known vm2 3.9.16 vulnerability via the code editor to get RCE as svc.
2. **Privesc technique in one sentence:** Cracked a hash from a local SQLite DB to become joshua, then abused an unquoted `[[ == ]]` pattern-match in a sudo-run bash script (plus pspy to capture the real root password) and reused that password to become root.
