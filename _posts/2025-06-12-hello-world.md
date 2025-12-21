---
title: "hello"
date: 2023-06-03 00:00:00 +0800
categories: [HackTheBox - Easy Machines]
tags: [HackTheBox CTFs]
---

#### Enumeration Start

I ran the below Nmap port scan.

```
ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.194 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
```

```
─$ nmap -p$ports -sC -sV 10.10.10.222
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-23 21:46 SAST
Nmap scan report for 10.10.10.222
Host is up (0.67s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-title: Welcome
|_http-server-header: nginx/1.14.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```


I opened the web application and got a page desscribing the site as a place to get email related ssupport. The below is found in the contact us area also
