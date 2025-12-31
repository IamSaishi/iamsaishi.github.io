---
title: "HackTheBox - Delivery"
date: 2025-12-31 00:00:00 +0800
categories: [HackTheBox - Easy Machines]
tags: [HackTheBox CTFs]
---


---

Delivery is an easy difficulty Linux machine that features the support ticketing system osTicket where it is possible by using a technique called TicketTrick, a non-authenticated user to be granted with access to a temporary company email. This permits the registration at Mattermost and the join of internal team channel. It is revealed through that channel that users have been using same password variant "PleaseSubscribe!" for internal access. In channel it is also disclosed the credentials for the mail user which can give the initial foothold to the system. While enumerating the file system we come across the Mattermost configuration file which reveals MySQL database credentials. By having access to the database a password hash can be extracted from Users table and crack it using the "PleaseSubscribe!" pattern. After cracking the hash it is possible to login as user root.

---

## 1. Enumeration Phase

#### Port and Service Enumeration

The first step is always to begin by enumerating the target. I normally `ping` the target first to confirm that the host is up, and then I run an `nmap` scan against the target to identify open services.

```shell
─$ nmap -p- -sV -sC -T4 --min-rate=1000 10.10.10.222
 
...
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
...
80/tcp   open  http    nginx 1.14.2
|_http-title: Welcome
|_http-server-header: nginx/1.14.2
8065/tcp open  http    Golang net/http server
|_http-title: Mattermost
...
```

After the initial scan, I note down all detected open services, including their version information.

**Services Identified:**
- **Port 22**      [SSH Service]    -  OpenSSH 7.9p1
- **Port 80**      [HTTP Service]  -  nginx 1.14.2
- **Port 8065**  [HTTP Service]  -  Golang net/http server

---

#### Web Application Enumeration - `Port 80`

After identifying the open services, we can see that two of them relate to HTTP web applications. Whenever I see this, I spin up Burp Suite and start interacting with the web application.

Navigating to `http://10.10.10.222:80` opens a landing page for a web application. It’s extremely simple, with very little information to gather.

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img1.png)

**Notable observations:**
- A `Contact Us` section
- A `Helpdesk` link pointing to the subdomain: `helpdesk.delivery.htb`

So I take note of the new subdomain and browse to it next.

---
#### Web Application Enumeration - `osTicket`

When navigating to the `helpdesk.delivery.htb` subdomain, I'm greeted with the below screen:

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img2.png)

This happens because there is no DNS record for `helpdesk.delivery.htb`, so we first need to add this subdomain, as well as the main domain, to `/etc/hosts`:

```shell
echo "10.10.10.222 helpdesk.delivery.htb delivery.htb" | sudo tee -a /etc/hosts 
```

After doing this, visiting `helpdesk.delivery.htb` loads the page correctly.

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img3.png)

This page appears to be a support ticketing system. It provides several useful details:

**Application Observations:**
- Shows I'm logged in as Guest User.
- Can open a New Ticket
- Can check Ticket status.
- Says `Powered by osTicket`. This gives us what application is behind this web application.

I decide to open a new ticket and see what happens. Navigating to the `Open a New Ticet` page, I'm greeted with the below page:

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img4.png)

There is a form we can fill in to create a ticket. I use fake test information to register. After successfully creating a ticket, we get the below message:

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img5.png)

**Key Observations:**
- I can check the ticket status on the status page
- The **Ticket ID is `5800033`**
- I can update the ticket by emailing `5800033@delivery.htb`

The last point is very interesting as it gives us slight control on a internal company email. The part mentioning we can update tickets using the given address is powerful as it means we can receive emails from that email which is an internal email.

***Important:** We don’t actually gain a real company mailbox. osTicket converts incoming email replies into ticket updates. This means any verification email sent to `5800033@delivery.htb` is appended directly to the ticket thread.* 

I then navigate to the `View Ticket` page and see the below page which is details about the ticket we created and we may be able to see any updates to our ticket:

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img6.png)

After reviewing the ticket page, I move on to explore the next open service from the initial scan.

---
#### Web Application Enumeration - `Mattermost`

Navigating to `http://10.10.10.222:8065`reveals a Mattermost instance. Mattermost is an open-source, self-hosted collaboration platform similar to Slack or Microsoft Teams, commonly used in IT and development environments.

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img7.png)

**Application Observations:**
- Users can create new accounts
- Users can log in if they already have credentials
- Users can reset passwords via **“I forgot my password”**

I decide to create an account using the internal email address we obtained earlier, along with some test details. The reason I use the temporary email we were given for our ticket is due to Mattermost not allowing public sign-ups globally.

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img8.png)

After registering, I receive a message stating the email address must be verified, meaning a verification email has been sent.


![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img9.png)

Lucky for me, I used the internal email we were given and any emails sent to the email address would be shown as updates on our ticket 🙌.  

Navigating back to the `View Ticket` page on the `osTicket` web application, we can see have a new post which contains an email activation link from `Mattermost` application. 

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img10.png)

We can just copy and paste the link that was sent and verify our email. After this, I head back to the Mattermost application and attempt to login with the credentials we used to register the account earlier

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img11.png)

Lessgo!! This Works!! 😁

---

#### Leaked Credentials in the Internal Messages

Once logged in, I gain access to the internal chat channels. While reviewing the messages, I discover credentials being shared internally:

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img12.png)


**Exposed Credentials**
- **Username:** `maildeliverer`
- **Password:** `Youve_G0t_Mail!`

There is also a discussion indicating users reuse passwords across services, with an example string: `PleaseSubscribe!`.

Another message mentions that although this password is not in **rockyou.txt**, attackers could use **hashcat rules** to generate variants and crack it. This suggests weak password hygiene internally.

This also hints at us using hashcat rules to create a custom password list based on the `PleaseSubscribe!` password.

---
## 2. Initial Access Phase
#### Using Exposed Credentials on SSH Service

Using the above credentials, I attempt to use them to log in to the server via the SSH Service:

```shell
ssh maildeliverer@10.10.10.222
```

Yessir!! 🕺

This gives me access to the server as the `maildeliverer` user. Within the users Current Working Directory, I find the a file containing the user flag.

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img13.png)


----

## 3. Post-Exploitation & Privilege Escalation Phase

#### Linux Privilege Escalation Enumeration

With a foothold on the box, the next goal is to escalate privileges to root. I began with the usual Linux privilege escalation checks. While most results weren’t interesting, I did spot the following user entry:

```shell
maildeliverer@Delivery:~$ cat /etc/passwd 

...
mysql:x:110:118:MySQL Server,,,:/nonexistent:/bin/false
...
```

Since there’s a `mysql` service account, I checked for local listening services:

```shell
maildeliverer@Delivery:~$ netstat -ltp

...

Proto Recv-Q Send-Q Local Address           Foreign Address         State               
tcp        0      0 localhost:mysql         0.0.0.0:*               LISTEN              ...        
```

Bingo!! So MySQL is running locally and only bound to `localhost`. I note this down and quickly finish up some other checks that resulted in nothing fruitful such as enumerating sudo rights for the current user and looking for any SUID binaries that we can use as an exploitation vector.

I tried logging in using credentials previously discovered earlier in the attack path, but those attempts were denied. After further enumeration, I located the Mattermost configuration file, which often stores database credentials:

```shell
cat opt/mattermost/config/config.json | grep DataSource
```

This revealed the datasource connection string:

```shell
maildeliverer@Delivery:/opt/mattermost/config$ cat config.json | grep DataSource
 
"DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
"DataSourceReplicas": [],
"DataSourceSearchReplicas": [],
```

**Exposed Credentials**
- **Username:** `mmuser`
- **Password:** `Crack_The_MM_Admin_PW`

Using the found credentials, I attempt to login to the MySQL service using these credentials:

```shell
mysql -u mmuser -pCrack_The_MM_Admin_PW
```

This gets me in! 😎

Inside the `mattermost` database, the `Users` table contained usernames and bcrypt password hashes. One of the entries corresponded to the `root` account:

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img14.png)

I make note of the `root` credentials

**Exposed Credentials**
- **Username:** `root`
- **Password Hash:** `$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO`

---

#### Password Mutation & Cracking

Earlier in the engagement, it was revealed that users reused variations of the password:

```
PleaseSubscribe!
```

To generate likely variations, I created a custom wordlist using Hashcat rule-based mutations:

```shell
echo PleaseSubscribe! | hashcat -r /usr/share/hashcat/rules/best64.rule --stdout > wordlist.txt
```

You take a *seed word*, which is `PleaseSubscribe!` in this case, and let Hashcat mutate it using rule files. This will create a custom wordlist based on the seed word. Once we have the wordlist we can just use `john the ripper` to crack the password hash.

---

#### Logging in as root user

With the wordlist ready, I used John the Ripper to attack the bcrypt hash:

```shell
└─$ john root_hash --wordlist=wordlist.txt                                
...
PleaseSubscribe!21 (?)     
...
```

John successfully cracked the hash!

This returns `PleaseSubscribe!21` as the password. 

I switched to the root user:

```shell
su root
```

Entered the cracked password, and obtained root-level access. From there, I retrieved the final flag from the `/root` directory

---
---

## Conclusion

I enjoyed this machine. Getting to the user flag wasn't that difficult but the root flag took more time as I didn't think to look for a configuration file at first. **😔**

On to the next machine!

![Description](assets/HTB-Delivery-CTF-Images/HTB - Delivery CTF - img15.png)

---
