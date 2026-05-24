---
title: "Internal - TryHackMe Writeup"
date: 2025-11-23
draft: false
summary: "Writeup for Internal CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "rce", "wordpress", "suid", "brute force", "jenkins"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/internal/cover.webp"
  caption: "Internal TryHackMe Challenge"
  alt: "Internal cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in three weeks.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/internal

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN internal.nmap
```

![performing an nmap scan on internal machine](https://cdn.ziomsec.com/internal/1.webp)

## Foothold

I mapped the domain *internal.thm* with the IP in my *host* file and accessed the web application through my browser.

![accessing the web applicaiton via browser](https://cdn.ziomsec.com/internal/2.webp)

I then used **ffuf** to fuzz for hidden directories and found a wordpress installation.

```shell
ffuf -u http://internal.thm/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![fuzzing hidden directories on the target](https://cdn.ziomsec.com/internal/3.webp)

The `/blog` endpoint also pointed towards **wordpress**

![accessing the wordpress instance](https://cdn.ziomsec.com/internal/4.webp)

Since, it was **wordpress**, I tried accessing the *wp-login* endpoint. I tried common username passwords but couldn't log in. The `/wordpress` endpoint pointed to a page that did not exist.

![accessing the wordpress instance](https://cdn.ziomsec.com/internal/5.webp)

I then went back to the `/blog` endpoint and viewed a blog that was posted. The author could be a valid user.

![trying to enumerate a valid user](https://cdn.ziomsec.com/internal/6.webp)

I then fuzzed for hidden files using **ffuf** but found nothing interesting.

So I switched back to the *wp-login* endpoint and verified if *admin* was a valid username.

![trying to enumerate a valid user](https://cdn.ziomsec.com/internal/7.webp)

With no other leads, I tried bruteforcing the *admin* password using **hydra** and found it.

```shell
hydra -l 'admin' -P /usr/share/wordlists/rockyou.txt internal.thm http-post-form "/blog/wp-login.php:log=^USER^&pwd=^PASS^:incorrect"
```

![bruteforcing the admin password with hydra](https://cdn.ziomsec.com/internal/8.webp)

I then logged in to access the **Wordpress** dashboard.

![logging into Wordpress](https://cdn.ziomsec.com/internal/9.webp)

With access to the **wordpress** dashboard, I could get a reverse shell. I referred to **hacktricks** and got a reverse shell by changing the *404.php* template with a **pentestmonkey**'s php reverse shell and calling the endpoint to trigger the payload.
- https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/wordpress.html?highlight=wordpress#panel-rce

![adding a reverse shell php payload](https://cdn.ziomsec.com/internal/10.webp)

```shell
curl http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

```shell
rlwrap nc -lnvp 443
```

![triggering the payload to get a reverse shell](https://cdn.ziomsec.com/internal/11.webp)

## Privilege Escalation

### Gaining Shell As `root` By Exploiting `pkexec`

After receiving the shell, I viewed the */home* directory and found a user called *aubreanna*. However, I did not have the privilege to view the contents inside it.

I then viewed the wordpress installation directory.

![enumerating wordpress directory](https://cdn.ziomsec.com/internal/12.webp)

The config file often contains sensitive information, so I viewed the *wp-config* file and found the **mysql** credentials.

```shell
cd /var/www/html/wordpress
cat wp-config.php
```

![finding sql credentials via config file](https://cdn.ziomsec.com/internal/13.webp)

I then logged into the server and looked at the contents present inside the *wordpress* database.

```shell
mysql -u wordpress -p
use wordpress;
show tables;
```

![logging into the mysql server](https://cdn.ziomsec.com/internal/14.webp)

However, I found nothing except the *admin* user's hash.

```shell
select * from wp_users;
```

![admin user hash](https://cdn.ziomsec.com/internal/15.webp)

I then looked for uncommon binaries with **SUID** bit set

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

I found **pkexec** and decided to give the **PwnKit** exploit a try. If it worked, I could directly get *root* access.

![listing files with suid bit](https://cdn.ziomsec.com/internal/16.webp)

I then downloaded the **PwnKit** exploit and gave it executable permissions.
- https://github.com/ly4k/PwnKit

Upon executing the exploit, I got root shell.

![exploiting pkexec for root access](https://cdn.ziomsec.com/internal/17.webp)

With root access, I could read the contents of both, user flag and root flag.

![capturing the user and root flag](https://cdn.ziomsec.com/internal/18.webp)

However, this privesc vector was unintended.

### Exploiting Jenkins

I explored other folders and found the credentials for *aubreanna* in a text file inside the */opt* directory : `bubbl3guM!@#123`

```shell
cat /opt/wp-save.txt
```

I logged in to the system as *aubreanna* using **ssh**.

```shell
ssh aubreanna@internal.thm
```

I found a note inside my home directory that said there was a jenkins service running internally on some IP.

![finding a note in the user directory](https://cdn.ziomsec.com/internal/19.webp)

I viewed my IPs and realized that the **jenkins** server was likely running on an internal machine and not locally.

```shell
ifconfig
```

![verifying target IP](https://cdn.ziomsec.com/internal/20.webp)

So, I performed a local port forward to be able to access **jenkins** hosted on the internal network from my local port 8888 through the compromised machine.

```shell
ssh -L 8888:172.17.0.2:8080 aubreanna@internal.thm
```

I then accessed **jenkins** through my browser.

![performing a local port forward to access internal machine](https://cdn.ziomsec.com/internal/21.webp)

I fuzzed for hidden directories and found a bunch of interesting endpoints.

```shell
ffuf -u http://localhost:8888/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![fuzzing hidden directories](https://cdn.ziomsec.com/internal/22.webp)

However, none of them contained anything special. They just threw a *404 not found* error.

![accessing the discovered directory](https://cdn.ziomsec.com/internal/23.webp)

With no other leads, I looked for default credentials and found a username called `admin`. I tried the username with common passwords but failed to log in.

I then brute forced the password using **hydra** from the **rockyou** wordlist.

```shell
hyda -l 'admin' -P /usr/share/wordlists/rockyou.txt localhost http-post-form '/j_acegi_security_check:j_username=^USER^&j_password=^PASS^:Invalid' -s 8888
```

![bruteforcing admin credentials for jenkins](https://cdn.ziomsec.com/internal/24.webp)

After logging in, I was lost. So I referred to **hackingarticles** and found a way to execute **groovy** scripts.
- https://www.hackingarticles.in/jenkins-penetration-testing/

![script console on jenkins](https://cdn.ziomsec.com/internal/25.webp)

I visited **revshells** and configured a **groovy** script that I could run on the **jenkins** server and receive a reverse shell on my **netcat** listener.
- https://www.revshells.com

![configuring a groovy script for reverse shell](https://cdn.ziomsec.com/internal/26.webp)

After executing the script, I received a reverse shell.

![receiving a reverse shell as jenkins](https://cdn.ziomsec.com/internal/27.webp)

I then explored the system and found root user's password inside a text file in the */opt* directory.

```shell
cat /opt/note.txt
```

![finding the root password in a text file](https://cdn.ziomsec.com/internal/28.webp)

With the *root* password, I could simply switch my user for a privileged access.

```shell
su root
```

![switching to root user](https://cdn.ziomsec.com/internal/29.webp)

That's it from my side!
Until next time :)

---