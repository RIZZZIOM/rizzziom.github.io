---
title: "Retro"
date: 2024-11-21
draft: false
summary: "Writeup for Retro CTF challenge on TryHackMe."
tags: ["windows", "kernel exploit", "rce", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/retro/cover.webp"
  caption: "Retro TryHackMe Challenge"
  alt: "Retro cover"
platform: "TryHackMe"
---

Can you time travel? If not, you might want to think about the next best thing.
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/r/room/retro

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and services running on the target.

```shell
nmap -A -p- TARGET -Pn -oN retro.nmap --min-rate 10000
```

![](https://cdn.ziomsec.com/retro/1.webp)

## Initial Foothold

The machine had an http service and rdp running on it. I started of by visiting the webpage through my browser.

![](https://cdn.ziomsec.com/retro/2.webp)

It was just a default landing page. So I performed directory bruteforce using **ffuf** to find other directories and files.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/retro/3.webp)

The **ffuf** scan revealed a page called `/retro`. So I visited it. This page looked like a blog on retro games.

![](https://cdn.ziomsec.com/retro/4.webp)

I fuzzed `/retro` using **ffuf** to find interesting files or directories.

```shell
ffuf -u http://TARGET/retro/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fs 0 -mc 200,302,301
```

![](https://cdn.ziomsec.com/retro/5.webp)

The **ffuf** scan discovered a **wordpress** login panel. I tried logging in using a random username and password and got an error saying *invalid username*. 

![](https://cdn.ziomsec.com/retro/6.webp)

This error mechanism made it easy to find valid users. I navigated to the `/retro`'s home page and looked for potential usernames. Since the blogs were written by **Wade**, I could try using it as a **username**. I clicked on **Wade** to find information about the author.

![](https://cdn.ziomsec.com/retro/7.webp)

I also tried using this username in the login panel. And received an error of *invalid password*.

![](https://cdn.ziomsec.com/retro/8.webp)

In the **author's** page, I found downloadable **RSS** data.

![](https://cdn.ziomsec.com/retro/9.webp)

I downloaded and viewed the data. One of the files revealed the password of **Wade**.

```shell
cat Zq1vjf2d
```

![](https://cdn.ziomsec.com/retro/10.webp)

I logged in and got access to the **wp-admin** panel.

![](https://cdn.ziomsec.com/retro/11.webp)

I referred to **hacktricks** for ways to get an RCE from wp-admin.
- https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/wordpress#panel-rce

> **note:** Since the target is a windows machine, I used a cross platform php-reverse-shell from the below github repo.

- https://github.com/ivan-sincek/php-reverse-shell

I added my reverse shell payload in the **404.php** template so that I could execute it by causing an error.

![](https://cdn.ziomsec.com/retro/12.webp)

After updating the php code, I started a reverse shell listener using **nc** and triggered the payload by trying to navigate to a non-existent path.

```shell
curl http://TARGET/retro/index.php/author/wade/dfds
```

This gave me a reverse shell.

![](https://cdn.ziomsec.com/retro/13.webp)

Alternatively, I also found out that **wade** reused his password. So the same credentials could also be used with **rdp**.

```shell
hydra -l 'wade' -p 'parzival' rdp://TARGET
```

![](https://cdn.ziomsec.com/retro/14.webp)

**rdp** would provide a graphical access to the system making interaction much easier. So I connected to the target using **xfreerdp**.

```shell
xfreerdp /u:wade /p:parzival /v:TARGET
```

On the **Desktop**, I found the **user.txt** file.

![](https://cdn.ziomsec.com/retro/15.webp)

## Privilege Escalation

The desktop had **chrome** so I opened it. It had a **cve** in bookmarks so I read about the vulnerability on my local system.
- https://nvd.nist.gov/vuln/detail/CVE-2019-1388

![](https://cdn.ziomsec.com/retro/16.webp)

This vulnerability allowed privilege escalation so I searched for POCs so that I could follow the steps to get administrator access. 

I found this **github** repo with a POC and tried following the steps.
- https://github.com/nobodyatall648/CVE-2019-1388

Here's the steps to reproduce the PoC
1) find a program that can trigger the UAC prompt screen
2) select "Show more details"
3) select "Show information about the publisher's certificate"
4) click on the "Issued by" URL link it will prompt a browser interface.
5) wait for the site to be fully loaded & select "save as" to prompt a explorer window for "save as".
6) on the explorer window address path, enter the cmd.exe full path: `C:\WINDOWS\system32\cmd.exe`
7) now you'll have an escalated privileges command prompt. 

The recycle bin contained an application that could be vulnerable. So I restored it and followed the steps from the POC.

![](https://cdn.ziomsec.com/retro/17.webp)

![](https://cdn.ziomsec.com/retro/18.webp)

![](https://cdn.ziomsec.com/retro/19.webp)

![](https://cdn.ziomsec.com/retro/20.webp)

![](https://cdn.ziomsec.com/retro/21.webp)

I got stuck here as I didn't get an option to choose a browser. 
- So I referred to https://muirlandoracle.co.uk/2020/01/06/retro-write-up/ and restarted the machine. 
- Before the exploit, I initialized both chrome and edge browsers. However, this time as well it didn't work. 
- Lastly, I manually navigated to `https://www.verisign.com/repository/CPS` but that didn't work aswell.

As a last resort, I tried looking for kernel exploits.

```shell
systeminfo
```

![](https://cdn.ziomsec.com/retro/22.webp)

I found a couple of exploits for the windows build running on the target. I visited the **`COM Aggregate Priv Exec`** page on **exploit-db**.
- https://www.exploit-db.com/exploits/42020

I looked for that particular cve's POC and found a **github** repo that contained a binary that could escalate my privileges.
- https://github.com/WindowsEploits/Exploits/tree/master/CVE-2017-0213

![](https://cdn.ziomsec.com/retro/23.webp)

So I downloaded the binary on my local system and transferred it to the target machine. Upon execution, the exploit spawned another instance of **cmd** as **administrator**.

![](https://cdn.ziomsec.com/retro/24.webp)

![](https://cdn.ziomsec.com/retro/25.webp)

I captured the **root** flag from **Administrator's** Desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt.txt
```

![](https://cdn.ziomsec.com/retro/26.webp)

## Closure

Here's a brief summary of how I pwned **retro**:
- **nmap** scan revealed a webserver running on the target along with **rdp**.
- Directory fuzzing revealed a gaming blog page and a **login** panel.
- Further reconnaissance on the `/retro` revealed a potential username `Wade`. Files present on Wade's profile also revealed his password.
- I then tried these credentials with **rdp** and got a graphical access on the system.
- I captured the user flag from Desktop.
- I tried exploiting the bookmarked cve but it failed due to a bug.
- I then exploited the kernel and got **administrative** access.
- Finally I captured the root flag from **administrator's** desktop.

That's it from my side !
Until next time :)

---