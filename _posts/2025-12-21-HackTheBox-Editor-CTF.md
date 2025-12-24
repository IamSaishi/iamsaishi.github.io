---
title: "HackTheBox - Editor"
date: 2023-12-21 00:00:00 +0800
categories: [HackTheBox - Easy Machines]
tags: [HackTheBox CTFs]
---
## Summary

Editor is an Easy Linux machine that has us looking for vulnerable services that we can use to get an initial foothold on the target. Through a outdated and vulnerable instance of XWiki, we are able to take advantage of this to get Remote Code Execution on the Target Server. Once we get a reverse shell, we can find configuration files for XWiki where one of them contains credentials. We can then reuse the credentials on another users to log in to the SSH service. Once getting in, we perform Linux Privilege Escalation checks and find we can perform a Untrusted search path vulnerability attack to gain Local privilege escalation to root. 

---
## 1. Enumeration Phase
#### Host Discovery

We start with enumerating the target. Before starting my service scans, I always run a `ping` request *(ICMP Echo Requests)* to the target. We can see this below.

```
└─$ ping 10.10.11.80                
PING 10.10.11.80 (10.10.11.80) 56(84) bytes of data.
64 bytes from 10.10.11.80: icmp_seq=1 ttl=63 time=188 ms
64 bytes from 10.10.11.80: icmp_seq=2 ttl=63 time=310 ms
```

We can see that we receive an *ICMP Echo Reply* from the target indicating that the target is 
reachable.

---
#### Port and Service Enumeration

I then perform a full TCP port scan, followed by service and version detection on discovered ports.

```
└─$ ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.80 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) 

└─$ nmap -p$ports -sC -sV 10.10.11.80
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-23 23:55 SAST
Nmap scan report for 10.10.11.80
Host is up (0.38s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
...
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editor.htb/
...
8080/tcp open  http    Jetty 10.0.20
...
|_Requested resource was http://10.10.11.80:8080/xwiki/bin/view/Main/
...
```

**Services Identified:**
- **Port 22**      [SSH Service]    -  OpenSSH 8.9p1
- **Port 80**      [HTTP Service]  -  nginx 1.18.0
- **Port 8080**  [HTTP Service]  -  Jetty 10.0.20

I first map all services and their detected versions identified during the port scan to gain a clear understanding of the exposed attack surface before manually interacting with the web applications.

---
#### Virtual Host Resolution

Before doing so, whenever a domain is referenced in the Nmap output, such as through an `http-title: Did not follow redirect to` message, I add the domain to the `/etc/hosts` file to ensure proper resolution during further testing.

```
└─$ echo "10.10.11.80 editor.htb" | sudo tee -a /etc/hosts  
10.10.11.80 editor.htb
```

---
#### Web Enumeration - `editor.htb`

With the domain now added to the `/etc/hosts` file, I proceed to examine the web application. While manually reviewing the site, I also begin directory fuzzing using `gobuster` to identify any hidden or unlinked resources.

```
└─$ gobuster dir -u http://editor.htb/ -w /usr/share/seclists/Discovery/Web-Content/common.txt

...
===============================================================
/assets               (Status: 301) [Size: 178] [--> http://editor.htb/assets/]
/index.html           (Status: 200) [Size: 631]
Progress: 4746 / 4746 (100.00%)
... 
```

The directory fuzzing doesn't yield much. Navigating to `http://editor.htb/assets/` gives a nginx error `403 Forbidden`. 

**Application Observations:**

- Landing page for **SimplistCode Pro**
- Download links for Debian/Ubuntu and Windows
- About page describing the application
- A **Docs** link present in the navigation bar

Clicking the **Docs** link in the navigation bar returns an error in the browser. Reviewing this request in the Burp Suite browser reveals a reference to a previously unknown subdomain.

![[Pasted image 20251224002758.png]]

---
#### Wiki Subdomain Discovery

This looks like a new subdomain we can add to `/etc/hosts` file.

```
echo "10.10.11.80 wiki.editor.htb" | sudo tee -a /etc/hosts
```

We can then begin to manually enumerate the `wiki.editor.htb` endpoint.

![[Pasted image 20251224004224.png]]

**Key Observations:**
- Full URL: `http://wiki.editor.htb/xwiki/bin/view/Main/`.
- Appears to be the **SimplistCode Pro documentation**.
- Login functionality available.
- `JSESSIONID` cookie set.
- XWiki version disclosed: **XWiki Debian 15.10.8**.
- Page metadata shows: _Last modified by Neal Bagwell_.
- Installation documentation available.
