---
title: "Silver Platter"
date: 2025-08-27
draft: false
summary: "Writeup for Silver Platter CTF challenge on TryHackMe."
tags: ["linux", "idor", "auth bypass", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/silver-platter/cover.webp"
  caption: "Silver Platter TryHackMe Challenge"
  alt: "Silver Platter cover"
platform: "TryHackMe"
---

Can you breach the server?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/silverplatter

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and services running on them.

```shell
nmap -A -p- TARGET --min-rate 1000 -oN silverplatter.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 8080     | http        |

![](https://cdn.ziomsec.com/silver-platter/1.webp)

## Initial Foothold

The **nmap** scan discovered a web server running on the target. So I accessed it using my browser.

![](https://cdn.ziomsec.com/silver-platter/2.webp)

The contact page revealed a username that could be used later.

![](https://cdn.ziomsec.com/silver-platter/3.webp)

I then moved onto the other port where an **http** proxy was running. I looked for directories using **ffuf** and found 2 new endpoints.

```shell
ffuf -u http://TARGET:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,302
```

![](https://cdn.ziomsec.com/silver-platter/4.webp)

One endpoint wasn't accessible, however the other one redirected us to a login panel.

![](https://cdn.ziomsec.com/silver-platter/5.webp)

I used the username `scr1ptkiddy` from the contact page and tried logging in using default credentials but failed. I also tried *Get a new password* but found no interesting functionality.

### Authentication Bypass

I then looked for vulnerabilities associated with the name of the cms 'silverpeas' and found an **authentication bypass** vulnerability.

![](https://cdn.ziomsec.com/silver-platter/6.webp)

I could bypass authentication by simply removing the password field. So I captured the login request using **burp suite** and removed the password field to log into the web app.

![](https://cdn.ziomsec.com/silver-platter/7.webp)

![](https://cdn.ziomsec.com/silver-platter/8.webp)

I had a notification of a message sent to me by my manager.

![](https://cdn.ziomsec.com/silver-platter/9.webp)

### Shell As Tim

I recalled reading about a **broken access control** vulnerability on the cms so gave it a try.

![](https://cdn.ziomsec.com/silver-platter/10.webp)

![](https://cdn.ziomsec.com/silver-platter/11.webp)

![](https://cdn.ziomsec.com/silver-platter/12.webp)

I managed to get the ssh credentials by exploiting **IDOR** vulnerability. I then used it to connect to the target using **ssh**.

```shell
ssh tim@TARGET
```

![](https://cdn.ziomsec.com/silver-platter/13.webp)

Finally, I captured the user flag.

![](https://cdn.ziomsec.com/silver-platter/14.webp)

## Privilege Escalation

### Shell As Tyler

Running the **id** command revealed that the user *tim* was part of the **adm** group. This group is used for monitoring purpose.

![](https://cdn.ziomsec.com/silver-platter/15.webp)

Hence my user would have access to the system **logs**. I looked at the authentication logs in */var/log/* directory and extracted passwords from it.

```shell
cat /var/log/auth* | grep -ai "password"
```

![](https://cdn.ziomsec.com/silver-platter/16.webp)

I checked if the password that I found was a valid user credential of `tyler` using **hydra**.

```shell
hydra -l 'tyler' -p '_Zd_zx7N823/' ssh://TARGET
```

![](https://cdn.ziomsec.com/silver-platter/17.webp)

I then logged in as Tyler.

```shell
ssh tyler@TARGET
```

![](https://cdn.ziomsec.com/silver-platter/18.webp)

### Shell As Root

I then viewed my `sudo` privs and found that I could run all commands.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/silver-platter/19.webp)

Hence I spawned a **bash** shell as root using **sudo** and captured the root flag from */root* directory.

```shell
sudo /bin/bash
cat /root/root.txt
```

![](https://cdn.ziomsec.com/silver-platter/20.webp)

That's it from my side!
Happy hacking :)

---
