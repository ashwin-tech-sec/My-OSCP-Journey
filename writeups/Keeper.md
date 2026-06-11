# Keeper — Easy — Linux

**Date:** 10 June 2026

---

## Tags
`#linux` `#web` `#request-tracker` `#default-creds` `#keepass` `#cve-2023-32784` `#puttygen` `#privesc`

---

## What I Know

| Item            | Detail                                                  |
| --------------- | ------------------------------------------------------- |
| Target          | keeper.htb / tickets.keeper.htb (10.129.229.41)         |
| OS              | Linux (Ubuntu)                                          |
| Open ports      | 22 (SSH), 80 (HTTP)                                      |
| Services        | OpenSSH 8.9p1; nginx 1.18.0                              |
| Web app         | Request Tracker (RT) 4.4.4                               |
| Initial creds   | RT admin via default creds `root:password`              |
| User            | lnorgaard (cleartext password in RT profile)            |
| Privesc vector  | KeePass dump (CVE-2023-32784) → root PuTTY SSH key       |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.229.41 -oA keeper_Scan_Initial

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-11 12:44 +0100
Nmap scan report for 10.129.229.41
Host is up (0.042s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 18.61 seconds
```
<img width="986" height="328" alt="image" src="https://github.com/user-attachments/assets/56f365d9-9626-44e8-b14a-b22076375378" />

```
nmap -sC -sV -Pn 10.129.229.41 -p22,80 -oA keeper_Scan_Detailed

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-11 12:45 +0100
Nmap scan report for 10.129.229.41
Host is up (0.016s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 35:39:d4:39:40:4b:1f:61:86:dd:7c:37:bb:4b:98:9e (ECDSA)
|_  256 1a:e9:72:be:8b:b1:05:d5:ef:fe:dd:80:d8:ef:c0:66 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.81 seconds
```

<img width="1226" height="481" alt="image" src="https://github.com/user-attachments/assets/579cfb80-e02b-4988-a26c-7679587478cc" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but I have no credentials yet, so this is a target to come back to once I find some.
- **Port 80 (HTTP)** — nginx 1.18.0 serving a web application. This is my entry point, so I'll focus here first.

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
OBSERVATION: Port 80 is open
WHAT IT TELLS ME: The server is hosting a web application
WHAT I AM GOING TO TRY: Navigate to the application and investigate it
WHY: To understand what the application does
RESULT: A single page that redirects to tickets.keeper.htb
WHAT NEXT:
- Add tickets.keeper.htb to /etc/hosts and investigate the page
- Directory fuzzing on the base IP
- Check for interesting JavaScript
- Check the page source
- Investigate the tech stack
- Inspect the HTTP response headers
```

<img width="878" height="356" alt="image" src="https://github.com/user-attachments/assets/fa27d948-6723-4bbd-9f31-6e053bf780d0" />

```
OBSERVATION:
- The application is Request Tracker (RT) 4.4.4+dfsg-2ubuntu1 (Debian)
- Nothing interesting in the JS, source, tech stack, or response headers
WHAT IT TELLS ME: An exploit or misconfiguration in RT may give me an entry point
WHAT I AM GOING TO TRY: Look for known RT vulnerabilities and test for default credentials
WHY: Ticketing systems are frequently left on default creds
RESULT: The default credentials root:password worked, logged into RT as the admin "root" account
WHAT NEXT: Explore the application for a path to a foothold
```

> **Note:** "root" here is the RT *application* admin account, not the system root user an important distinction.

<img width="1889" height="554" alt="image" src="https://github.com/user-attachments/assets/e2bbed73-205c-4f8c-87c0-42644f74ba13" />


<img width="790" height="140" alt="image" src="https://github.com/user-attachments/assets/1e0844b8-6026-4fd1-8815-68b4b69fce95" />

<img width="2558" height="885" alt="image" src="https://github.com/user-attachments/assets/e3353813-6b41-4753-a7e7-b56a1677cc4b" />

```
OBSERVATION: RT has many tabs to investigate
WHAT IT TELLS ME: Something in the app may be abusable for a foothold
WHAT I AM GOING TO TRY: Browse each tab for anything useful
RESULT: Under Admin > Users I found a second user, lnorgaard, and their profile/comment field contained a cleartext password
WHAT NEXT:
- Try the password in the RT application
- Try reusing it for SSH
```

<img width="1465" height="549" alt="image" src="https://github.com/user-attachments/assets/75eb2195-2f2b-49db-816e-f353ec54afa2" />

<img width="2036" height="886" alt="image" src="https://github.com/user-attachments/assets/7c388a35-3f83-4a7a-9b80-1fae174a1fcb" />

```
OBSERVATION:
- A ticket titled "Issue with Keepass Client on Windows" discusses a KeePass crash/dump; the app content is partly in Danish
- The lnorgaard password also worked for SSH,m logged in and found user.txt plus a ZIP file in the home directory
WHAT IT TELLS ME: The ticket references a KeePass memory dump. Since this involves an admin's KeePass database, recovering its master password could expose further credentials, potentially root's
WHAT I AM GOING TO TRY: Locate the KeePass dump and see whether the master password can be recovered from it
RESULT:
- The dump wasn't downloadable from inside RT
- After SSH'ing in I found the ZIP in lnorgaard's home directory
- KeePass 2.x is vulnerable to CVE-2023-32784, which recovers the master password from a memory dump
WHAT NEXT:
- Investigate the rest of the system
- Transfer the ZIP to my attack machine and examine it
```

<img width="2559" height="626" alt="image" src="https://github.com/user-attachments/assets/9881e1f4-ece7-4f60-aea1-a0d6d3f2013f" />

<img width="2132" height="1156" alt="image" src="https://github.com/user-attachments/assets/968dac2f-bb57-4570-b880-5578903ac3f3" />

<img width="936" height="330" alt="image" src="https://github.com/user-attachments/assets/cc9cf18a-0ed1-4f80-964d-07c74eb354df" />

```
OBSERVATION:
- Nothing else interesting on the system from general enumeration
- The ZIP contains two files: KeePassDumpFull.dmp and passcodes.kdbx
WHAT IT TELLS ME: I can recover the KeePass master password from the .dmp using CVE-2023-32784 via the PoC at https://github.com/vdohney/keepass-password-dumper
WHAT I AM GOING TO TRY: Run the dumper against KeePassDumpFull.dmp
RESULT:
- I had to edit keepass_password_dumper.csproj to target .NET SDK 6 to fix a build error
- The tool recovered: "M}dgrød med fløde", that did NOT open passcodes.kdbx
WHAT NEXT:
- Work out why the recovered password fails
- Open passcodes.kdbx
```

<img width="552" height="150" alt="image" src="https://github.com/user-attachments/assets/f035ba99-48a6-4fc6-b330-82169bea3b4b" />

<img width="1117" height="252" alt="image" src="https://github.com/user-attachments/assets/d8d825b8-1fbf-440a-9955-c688041d988a" />

<img width="1395" height="310" alt="image" src="https://github.com/user-attachments/assets/69dea873-6b4a-474c-a582-f4d464204dcf" />

<img width="1019" height="548" alt="image" src="https://github.com/user-attachments/assets/93d4aa2c-f129-4091-a6b5-c63aff041fcf" />

<img width="880" height="455" alt="image" src="https://github.com/user-attachments/assets/272c1846-2511-4739-88a8-f484b2a3f761" />

```
OBSERVATION:
- CVE-2023-32784 cannot recover the FIRST character of the master password, that is a documented limitation of the attack, not an error in my run
- The recovered fragment "dgrød med fløde" looked like part of a real Danish phrase. Searching led me to "rødgrød med fløde", a well-known Danish dessert (and pronunciation tongue-twister)
- So the true master password is rødgrød med fløde the missing leading character is "r"
WHAT IT TELLS ME: I had misread the unrecoverable first character as "M}". The fix is to substitute the correct leading "r" inferred from the Danish phrase
WHAT I AM GOING TO TRY: Open passcodes.kdbx with rødgrød med fløde
RESULT: The database unlocked
WHAT NEXT: Examine passcodes.kdbx for credentials usable to reach root
```

<img width="1093" height="371" alt="image" src="https://github.com/user-attachments/assets/c41a913e-57c1-4205-a3f0-2db96885d42f" />

<img width="994" height="792" alt="image" src="https://github.com/user-attachments/assets/331b69b1-1e58-4c0e-ad44-90cc41f0e9c5" />

```
OBSERVATION: Inside the KeePass database the root entry's "password" wasn't a normal password which can be used to SSH, the Notes field held a PuTTY-format SSH private key (PuTTY-User-Key-File)
WHAT IT TELLS ME: Password login as root failed, but I can authenticate with this key once it's converted from PuTTY (.ppk) format to an OpenSSH private key
WHAT I AM GOING TO TRY: Convert the PuTTY key to OpenSSH format with puttygen, then SSH in as root
WHY: SSH won't accept a PuTTY-format key directly; it needs an OpenSSH private key
RESULT: Converted the key to id_rsa with puttygen and logged in as root with ssh -i id_rsa root@10.129.229.41 — read root.txt
WHAT NEXT: convert key to id_rsa with puttygen and try to login as root
```

<img width="899" height="244" alt="image" src="https://github.com/user-attachments/assets/3b2c7091-1608-4669-b946-955aeb93f5c3" />


```
OBSERVATION: Converted the PuTTY key to an OpenSSH private key (id_rsa) with puttygen
WHAT IT TELLS ME: I can now authenticate to SSH as root using that key
WHAT I AM GOING TO TRY: ssh -i id_rsa root@10.129.229.41
RESULT: Logged in as root and read root.txt
```

<img width="1178" height="722" alt="image" src="https://github.com/user-attachments/assets/2e98570b-d9fd-429f-a882-e9aa1fbf9b90" />

<img width="1620" height="775" alt="image" src="https://github.com/user-attachments/assets/de343024-631f-4a64-8a52-496d9263791d" />

---

## Foothold

**How I got in:** The site redirects to a Request Tracker (RT) instance at tickets.keeper.htb. RT was left on the well-known default administrator credentials `root:password`, granting full admin access to the ticketing application. Inside, the user **lnorgaard**'s profile stored a cleartext password, which was reused for the system account — so the same password gave SSH access as lnorgaard and the user flag.

**Command / exploit used:**
```
# add vhost
echo "10.129.229.41  keeper.htb tickets.keeper.htb" | sudo tee -a /etc/hosts
# RT login: root:password  →  Admin > Users > lnorgaard  →  cleartext password
ssh lnorgaard@10.129.229.41      # password reused from RT profile
```

**Why it worked:** Two classic misconfigurations chained together. a default admin credential left unchanged on an internet-facing ticketing app, and password reuse between the application profile and the system account.

---

## Privilege Escalation

**Path taken (lnorgaard → root):** lnorgaard's home directory held a ZIP containing `KeePassDumpFull.dmp` and `passcodes.kdbx` a memory dump of a KeePass 2.x process and its database. KeePass 2.x is vulnerable to **CVE-2023-32784**, which recovers the master password from such a dump (all characters except the first). The dumper returned `?dgrød med fløde`; recognising it as the Danish phrase **rødgrød med fløde**, I supplied the missing leading "r" and unlocked the database. Inside, the root entry stored a **PuTTY-format SSH private key**. I converted it to an OpenSSH key with `puttygen` and logged in as root.

```
# recover master password (first char unrecoverable by design)
dotnet run --project keepass-password-dumper -- KeePassDumpFull.dmp
# open passcodes.kdbx with: rødgrød med fløde
# export the root PuTTY key from KeePass, then:
puttygen root.ppk -O private-openssh -o id_rsa
chmod 600 id_rsa
ssh -i id_rsa root@10.129.229.41
```

**Why it worked:** A sensitive KeePass memory dump was left readable in a user's home directory, and the installed KeePass version was vulnerable to master-password recovery. The database then stored a reusable root SSH key, turning database access directly into root access. The only real "trick" was inferring the unrecoverable first character from a recognisable Danish phrase.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Logged into Request Tracker with default creds `root:password`, pulled a user's cleartext password from their profile, and reused it for SSH.
2. **Privesc technique in one sentence:** Recovered a KeePass master password from a memory dump via CVE-2023-32784, opened the database, extracted a root PuTTY SSH key, converted it with puttygen, and logged in as root.
