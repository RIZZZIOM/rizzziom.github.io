---
title: "Root Me"
date: 2024-03-31
draft: false
summary: "Writeup for Root Me CTF challenge on TryHackMe."
tags: ["linux", "file upload", "rce", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/rootme/cover.webp"
  caption: "Root Me TryHackMe Challenge"
  alt: "Root Me cover"
platform: "TryHackMe"
---

A ctf for beginners, can you root me?
<!--more-->
To access the lab, click on the link given below:-
- https://tryhackme.com/r/room/rrootme

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and services running on them. It also ran default **nse** scripts and displayed the results.

```shell
nmap -A -p- TARGET -oN rootme.nmap --min-rate 10000 -Pn
```

![](https://cdn.ziomsec.com/rootme/1.webp)

## Foothold

The **nmap** scan revealed an **http** server running on port 80. So I accessed it from my browser.

![](https://cdn.ziomsec.com/rootme/2.webp)

The web page didn't reveal anything interesting so I used **ffuf** to find hidden directories.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/rootme/3.webp)

The directory bruteforce revealed a few directories. I accessed the `/css/` directory. It contained a **css** file for another page called **panel**. **Ffuf** had also discovered this directory. So I accessed `/panel` next.

![](https://cdn.ziomsec.com/rootme/4.webp)

This seemed like a **file upload** functionality. I created and uploaded a dummy file to try it out.

```shell
echo 'hello root' > test.txt
```

![](https://cdn.ziomsec.com/rootme/5.webp)

The directory bruteforce had revealed `/uploads` directory earlier. So I checked it to see if my file was uploaded.

![](https://cdn.ziomsec.com/rootme/6.webp)

![](https://cdn.ziomsec.com/rootme/7.webp)

After confirming the upload functionality, I used the **php pentestmonkey** payload to get a reverse shell. I navigated to **revshells** to first configure a payload that would get me a reverse shell.
- https://www.revshells.com

I entered my IP and port and saved it in a file called **`revshell.php`**

![](https://cdn.ziomsec.com/rootme/8.webp)

It blocked my php file. So I looked for ways to bypass this security mechanism and tried a few ways given on **hacktricks**.
- https://book.hacktricks.xyz/pentesting-web/file-upload

I changed the extension of my code to `Php`, `php%20` and finally managed to bypass the security check using **`.php5`**. After the upload was successful, I started my **netcat** listener.

![](https://cdn.ziomsec.com/rootme/9.webp)

![](https://cdn.ziomsec.com/rootme/10.webp)

I then navigated to the `/uploads` folder and clicked on my payload to execute it and got a reverse shell. I spawned a **pty** shell and captured the first flag from `/var/www` directory.

```shell
export TERM=xterm
python -c 'import pty;pty.spawn("/bin/bash")'
cat /var/www/user.txt
```

![](https://cdn.ziomsec.com/rootme/11.webp)

## Privilege Escalation

When I checked the binaries with **suid** bit, I found **python** which seemed uncommon.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/rootme/12.webp)

I visited **gtfobins** and looked for a way to exploit this misconfiguration for a privileged access.
- http://gtfobins.github.io/gtfobins/python/#suid

```
python -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

![](https://cdn.ziomsec.com/rootme/13.webp)

After becoming the **root** user, I had complete control over the system. So I navigated to `/root` directory and captured the final flag.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/rootme/14.webp)

## Closure

Here's a short summary of how I pwned **root me**:
- I discovered a directory allowing file upload operations by performing a directory brute force attack using **ffuf**.
- I tried uploading a php reverse shell payload but failed due to the target's security configuration.
- I bypassed the file extension check using **`.php5`** extension and executed the reverse shell payload to gain initial access.
- I captured the first flag from `/var/www`.
- I discovered **python** in the programs that has an **suid** bit which was very uncommon.
- I used **gtfobins** to find a way to exploit this misconfiguration and become **root**.
- I then captured the final flag from `/root` directory.

That's it from my side!
Until next time ;)

---