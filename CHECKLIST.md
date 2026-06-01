# ✅ Methodology Checklist

My go-to process for every box. Work top to bottom, take notes and screenshots as I go.

---

## 1. Recon
- [ ] Quick TCP scan to find open ports — `nmap -sC -sV -oN nmap-quick <IP>`
- [ ] Full port scan in the background — `nmap -p- -oN nmap-full <IP>`
- [ ] UDP scan on top ports — `nmap -sU --top-ports 20 <IP>`
- [ ] Note every service + version
- [ ] Add the box to my `/etc/hosts` if it uses a hostname

## 2. Service Enumeration
- [ ] Run `searchsploit` on every service version found
- [ ] **Web (80/443/etc.)**
  - [ ] Browse the site, view page source, check `robots.txt`
  - [ ] Directory/file brute force — `feroxbuster` / `gobuster`
  - [ ] Identify tech stack (Wappalyzer, headers, `whatweb`)
  - [ ] Look for login pages, upload forms, params to test
  - [ ] Vhost / subdomain fuzzing if a hostname is in play
- [ ] **SMB (139/445)** — `smbclient -L`, `enum4linux-ng`, check null sessions
- [ ] **FTP (21)** — try anonymous login
- [ ] **SSH (22)** — note version, no brute forcing on the exam
- [ ] **DNS (53)** — attempt zone transfer
- [ ] **Other ports** — research anything unusual

## 3. Foothold (Initial Access)
- [ ] Default / weak / guessable credentials
- [ ] Public exploit for an identified version
- [ ] Web attacks — LFI / RFI, SQLi, command injection, file upload bypass
- [ ] Exposed config files, backups, or secrets
- [ ] Get a stable reverse shell and upgrade it (`python3 -c 'import pty...'`)
- [ ] **Grab `local.txt` / `user.txt`**

## 4. Privilege Escalation
- [ ] Establish situational awareness — `whoami`, `id`, `hostname`, OS version
- [ ] **Linux**
  - [ ] `sudo -l`
  - [ ] SUID/SGID binaries — `find / -perm -4000 2>/dev/null`
  - [ ] Cron jobs, writable scripts, PATH abuse
  - [ ] Run `linpeas.sh`
  - [ ] Check GTFOBins for any useful binary
- [ ] **Windows**
  - [ ] Run `winPEAS`
  - [ ] Service misconfigs, unquoted service paths
  - [ ] Token privileges (`whoami /priv`)
  - [ ] AlwaysInstallElevated, stored creds, scheduled tasks
- [ ] Kernel / OS version exploits as a last resort
- [ ] **Grab `root.txt` / `proof.txt`**

## 5. Post-Exploitation & Notes
- [ ] Screenshot proof (`id` / `whoami` + the flag)
- [ ] Save every command that worked into my box notes
- [ ] Note anything new I learned for the [README](README.md) and CVE writeups
- [ ] Clean up any files I dropped on the target

---

> **Exam reminders:** screenshot *everything*, no Metasploit beyond the one allowed use, no automated exploit tools (sqlmap, autorecon, etc.) on the exam targets.
