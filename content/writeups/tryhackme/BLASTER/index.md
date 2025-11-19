---
title: "Blaster"
date: 2025-10-01
draft: false
summary: "Writeup for Blaster CTF challenge on TryHackMe."
tags: ["windows", "hardcoded creds", "uac bypass"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/blaster/cover.webp"
  caption: "Blaster TryHackMe Challenge"
  alt: "Blaster cover"
platform: "TryHackMe"
---

A blast from the past!
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/blaster

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN blaster.nmap -Pn
```

![](https://cdn.ziomsec.com/blaster/1.webp)

## Initial Foothold

The nmap scan revealed a web application running on port 80, so I accessed it on my browser.

![](https://cdn.ziomsec.com/blaster/2.webp)

It was a default IIS landing page so I fuzzed for hidden directories and found one called *retro*.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/blaster/3.webp)

I visited the directory and found a blogging application.

![](https://cdn.ziomsec.com/blaster/4.webp)

The author of the blog could be a user in the system so I kept note of it.

![](https://cdn.ziomsec.com/blaster/5.webp)

A comment on one of the blogs had a potential password.

![](https://cdn.ziomsec.com/blaster/6.webp)

I checked if the username and password were valid and found that I could use them to access the target through **rdp**.

```shell
hydra -l wade -p parzival TARGET rdp
```

![](https://cdn.ziomsec.com/blaster/7.webp)

I used **xfreerdp** to access the machine.

```shell
xfreerdp /u:'Wade' p: 'parzival' /v:TARGET
```

![](https://cdn.ziomsec.com/blaster/8.webp)

I found user.txt in the Desktop.

![](https://cdn.ziomsec.com/blaster/9.webp)

## Privilege Escalation

The Desktop had another application called hhupd.

![](https://cdn.ziomsec.com/blaster/10.webp)

I searched online for exploits and found articles for privilege escalation through UAC bypass.
- https://justinsaechao23.medium.com/cve-2019-1388-windows-certificate-dialog-elevation-of-privilege-4d247df5b4d7

I followed the steps given in the article to escalate my privilege to administrator:
- Right click on the application and select run as Administrator
- A dialogue box appears. Click the *show information...* link.

![](https://cdn.ziomsec.com/blaster/11.webp)

- A web browser will automatically open after clicking on the hyperlink. If not, click on the link again. The url should be of Verisign.

![](https://cdn.ziomsec.com/blaster/12.webp)

![](https://cdn.ziomsec.com/blaster/13.webp)

- Save the page in the following way:
	- click on save
	- ignore any warnings
	- a dialogue box will appear 
	- within the file explorer, navigate to `C:\Windows\System32` and open `cmd.exe`

![](https://cdn.ziomsec.com/blaster/14.webp)

![](https://cdn.ziomsec.com/blaster/15.webp)

![](https://cdn.ziomsec.com/blaster/16.webp)

- I spawned a shell as NT Authority System.

![](https://cdn.ziomsec.com/blaster/17.webp)

- I then captured the root flag from Administrator's Desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt
```

![](https://cdn.ziomsec.com/blaster/18.webp)

I wanted to get a meterpreter shell. So I started the **metasploit** framework.

```shell
msfdb run
```

I used the following exploit to get a reverse meterpreter shell by executing a payload on the target:

```shell
use exploit/multi/script/web_delivery
```

![](https://cdn.ziomsec.com/blaster/19.webp)

![](https://cdn.ziomsec.com/blaster/20.webp)

![](https://cdn.ziomsec.com/blaster/21.webp)

## Closure

Here's a short summary of how I solved **Blaster**:
- I discovered a blogging application on the IIS web server
- I found a username and password in the blogs
- I captured user.txt from Wade's Desktop
- I found an application called **hhupd** on the Desktop and looked for related exploits.
- I used the application to bypass UAC and spawn a shell as NT Authority System
- I migrated the shell to a meterpreter session using the `web_delivery` exploit in **metasploit** framework.
- I captured the root flag from Administrator's Desktop.

That's it from my side!
Until next time :)

---
