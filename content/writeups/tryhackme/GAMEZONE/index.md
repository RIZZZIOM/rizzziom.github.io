---
title: "Gamezone"
date: 2025-09-22
draft: false
summary: "Writeup for Gamezone CTF challenge on TryHackMe."
tags: ["linux", "sqli", "rce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/gamezone/cover.webp"
  caption: "Gamezone TryHackMe Challenge"
  alt: "Gamezone cover"
platform: "TryHackMe"
---

This room will cover SQLi, cracking a users hashed password, using SSH tunnels to reveal a hidden service and using a metasploit payload to gain root privileges.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/gamezone

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports, services running and run default **nse** scripts.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN gamezone.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/gamezone/1.webp)

## Foothold

I accessed the web application through my browser.

![](https://cdn.ziomsec.com/gamezone/2.webp)

I did a google reverse image search to find the name of the backgound character: `agent47`. This could be a valid username.

I then bypassed the login mechanism using **sql** injection.

![](https://cdn.ziomsec.com/gamezone/3.webp)

This gave me access to a page with search functionality.

![](https://cdn.ziomsec.com/gamezone/4.webp)

I searched for something and captured the request on **Burp Suite**.

![](https://cdn.ziomsec.com/gamezone/5.webp)

I wanted to test this for **SQL** injection as the login panel was also vulnerable to it. So, I added an asterisk after the value of `searchItem`.

![](https://cdn.ziomsec.com/gamezone/6.webp)

I then saved the request to a file and used **sqlmap** to dump database information.

![](https://cdn.ziomsec.com/gamezone/7.webp)

```shell
sqlmap -r request.txt --dbs
```

![](https://cdn.ziomsec.com/gamezone/8.webp)

After finding the database information, I dumped its contents.

```shell
sqlmap -r request.txt --dump --dbms=mysql -D db
```

![](https://cdn.ziomsec.com/gamezone/9.webp)

I found the password hash for the user **agent47**.

![](https://cdn.ziomsec.com/gamezone/10.webp)

I then cracked the hash on **Crackstation**.

![](https://cdn.ziomsec.com/gamezone/11.webp)

Alternately, the hash could also be cracked locally using john. 

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-sha256 hash
```

![](https://cdn.ziomsec.com/gamezone/12.webp)

After getting the credentials, I accessed the target using **ssh**.

```shell
ssh agent47@TARGET
```

![](https://cdn.ziomsec.com/gamezone/13.webp)

Finally, I captured the user flag from **agent47**'s home directory.

![](https://cdn.ziomsec.com/gamezone/14.webp)

## Privilege Escalation

I then used **netstat** to view connections and found that port 10000 was on listening state.

```shell
netstat -antp
```

![](https://cdn.ziomsec.com/gamezone/15.webp)

I performed a local port forwarding on that port and performed an **nmap** scan.

```shell
ssh -L 10000:TARGET:10000 agent47@TARGET
```

```shell
nmap -sV -p 10000 127.0.0.1
```

![](https://cdn.ziomsec.com/gamezone/16.webp)

Since it was running http, I tried accessing on my browser. However, access for the IP that I used while forwarding the port was denied.

![](https://cdn.ziomsec.com/gamezone/17.webp)

So I reforwarded the port using `Localhost` instead of using the LAN IP of the target.

```shell
ssh -L 10000:localhost:10000 agent47@TARGET
```

I then accessed the service running on port 10000 through my browser.

![](https://cdn.ziomsec.com/gamezone/18.webp)

I tried logging in using the **ssh** credential of the user '**agent47**'.

![](https://cdn.ziomsec.com/gamezone/19.webp)

I used **searchsploit** to look for exploits related to the **webmin** version running on the target.

```shell
searchsploit 'webmin 1.580'
```

![](https://cdn.ziomsec.com/gamezone/20.webp)

I looked at the **webmin** service and found that it was being run as **root**.

```shell
ps -aux | grep webmin
```

![](https://cdn.ziomsec.com/gamezone/21.webp)

So, if I exploited the service, I could gain root access. Hence, I started **metasploit** and selected the exploit related to the **webmin** version.

```shell
use exploit/unix/webapp/webmin_show_cgi_exec
set RHOSTS 127.0.0.1
set USERNAME agent47
set PASSWORD videogamer124
set SSL false
```

I also configured the payload

![](https://cdn.ziomsec.com/gamezone/22.webp)

After rechecking the configuration, I ran the exploit.

```shell
run
```

![](https://cdn.ziomsec.com/gamezone/23.webp)

I got a shell as **root** and captured the root flag from `/root` directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/gamezone/24.webp)

That's it from my side! Until next time ;)

---
