---
title: "Ultratech"
date: 2024-02-28
draft: false
summary: "Writeup for Ultratech CTF challenge on TryHackMe."
tags: ["linux", "command injection", "suid", "misconfigured service"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/ultratech/cover.webp"
  caption: "Ultratech TryHackMe Challenge"
  alt: "Ultratech cover"
platform: "TryHackMe"
---

You have been contracted by UltraTech to pentest their infrastructure. It is a grey-box kind of assessment, the only information you have is the company's name and their server's IP address.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/ultratech1

## Reconnaissance

I performed an **nmap** aggressive scan on the target to discover open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN ultratech.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 8081     | http        |
| 31331    | http        |

![](https://cdn.ziomsec.com/ultratech/1.webp)

## Initial Foothold

The **nmap** scan revealed http service running on port 8081 and port 31331. To find out more about these services, I accessed them through my browser.

![](https://cdn.ziomsec.com/ultratech/2.webp)

![](https://cdn.ziomsec.com/ultratech/3.webp)

Port 8081 seemed to host an API. This API was probably being used by the service running on port 31331.  I performed a directory bruteforce on the web page using **ffuf** and found a bunch of pages.

```shell
ffuf -u http://TARGET:31331/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/ultratech/4.webp)

I accessed *robots.txt* and found a link to another page.

![](https://cdn.ziomsec.com/ultratech/5.webp)

The sitemaps page revealed a few more endpoints.

![](https://cdn.ziomsec.com/ultratech/6.webp)

Upon accessing the *partners.html* page, I landed on a login panel.

![](https://cdn.ziomsec.com/ultratech/7.webp)

I tried logging in using default credentials but failed. I still had the API service so I performed a directory bruteforce on it to reveal 2 more endpoints.

```shell
ffuf -u http://TARGET:8081/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
```

![](https://cdn.ziomsec.com/ultratech/8.webp)

The source code of the login panel also contained an interesting file called *api.js*.

![](https://cdn.ziomsec.com/ultratech/9.webp)

This file contained the logic behind the authentication process of the login panel. It first pinged the API service to check its availability through the `/ping` endpoint. It then authenticated the credentials through the `/auth` endpoint.

![](https://cdn.ziomsec.com/ultratech/10.webp)

I tried accessing the `/ping` endpoint to get some more information. This just revealed the path where the web application was running.

![](https://cdn.ziomsec.com/ultratech/11.webp)

Then, following the code from *api.js*, I made a request to the */ping* endpoint to ping the server.

```shell
curl http://TARGET:8081/ping?ip=TARGET
```

![](https://cdn.ziomsec.com/ultratech/12.webp)

The output made it seem as an os command execution in the backend, so I tried chaining another command to the request. I was able to execute commands on the server. I found an **sqlite** file containing credentials of 2 users, `r00t` and `admin`.

```shell
curl 'http://TARGET:8081/ping?ip=TARGET`ls`'
curl 'http://TARGET:8081/ping?ip=TARGET`cat+utech.db.sqlite`'
```

![](https://cdn.ziomsec.com/ultratech/13.webp)

I cracked the hash using **crackstation** and saved them for later use.
- https://crackstation.net

```
r00t : n100906
admin : mrsheafy
```

I then verified if the credentials were valid for other services running on the target: **ftp** and **ssh**.

![](https://cdn.ziomsec.com/ultratech/14.webp)

The `r00t` credentials worked however the `admin` credentials failed. I then logged into the web application as admin and found a message.

![](https://cdn.ziomsec.com/ultratech/15.webp)

Then I got shell access on the target machine using **ssh**.

```shell
ssh r00t@TARGET
```

![](https://cdn.ziomsec.com/ultratech/16.webp)

## Privilege Escalation

### Exploiting `pkexec` For Root Acccess

Upon access, I looked for files under different user directories in the */home* directory but found nothing interesting. I then looked for programs running with **suid** bit and found **pkexec**. **Pkexec** is used by **Pollkit** to execute commands and was found to be vulnerably few years back. Since the machine was old, this version was likely vulnerable.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/ultratech/17.webp)

I navigated to the **PwnKit** exploit page on github and downloaded it on my local system.
- https://github.com/ly4k/PwnKit

I then downloaded the exploit from my local system onto the target system and provided execution rights to it. Upon executing the exploit, I got root access.

![](https://cdn.ziomsec.com/ultratech/18.webp)

I navigated to the */root* directory and found a message inside **private.txt**.

```shell
cat /root/private.txt
```

![](https://cdn.ziomsec.com/ultratech/19.webp)

### Exploiting Process Running As Root

An alternate way to escalate privileges was by viewing processed running as root.  Using **ps**, I listed all process and filtered the ones running as root. Here I found **docker**.

```shell
ps -aux | grep root
```

![](https://cdn.ziomsec.com/ultratech/20.webp)

I visited https://gtfobins.github.io/gtfobins/docker/ and found a way to break the shell restrictions and get root access.  I however, modified the command a bit and used this to get root access:

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt bash
```
- This command uses the `bash` image (or a more typical Debian-based image, assuming the default image has bash installed).
- The `bash` shell is invoked inside the `chroot` environment.

![](https://cdn.ziomsec.com/ultratech/21.webp)

## Closure

Here's a high level summary of how I pwned **UltraTech**:
- I abused the os command execution functionality and injected my own command to get user credentials.
- I used the credentials to get shell access on the target.
- I was able to get root access by 2 different methods:
	- Exploiting suid bit on **pkexec** using **PwnKit**.
	- Exploiting **docker** running as root.

That's it from my side! Until next time.

---