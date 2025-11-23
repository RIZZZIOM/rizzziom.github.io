---
title: "Daily Bugle"
date: 2024-02-14
draft: false
summary: "Writeup for Daily Bugle CTF challenge on TryHackMe."
tags: ["linux", "hash crack", "rce", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/daily-bugle/cover.webp"
  caption: "Daily Bugle TryHackMe Challenge"
  alt: "Daily Bugle cover"
platform: "TryHackMe"
---

Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum.
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/r/room/dailybugle

## Reconnaissance

I ran an **nmap** aggressive scan on the target to find open ports and the services running on them. It also performed a default script scan for additional information.

```shell
nmap -A -p- TARGET -oN bugle.nmap --min-rate 10000 -Pn
```

![](https://cdn.ziomsec.com/daily-bugle/1.webp)

## Foothold

The **nmap** scan revealed an **http** server on port **80** so I accessed it from my browser.

![](https://cdn.ziomsec.com/daily-bugle/2.webp)

This seemed like a simple blog. The *Login Form* seemed interesting but before moving forward, I decided to check out other things. The **nmap** scan had also revealed a **robots.txt** file with a few directory paths so I decided to check that out.

![](https://cdn.ziomsec.com/daily-bugle/3.webp)

Out of all the paths, only the **`/administrator`** path revealed a **login** page. Rest all were just blank pages.

![](https://cdn.ziomsec.com/daily-bugle/4.webp)

After that, I ran an **nse** script on the server to find the answer to find more information. I ran the *http-enum.nse* script and found a file which contained the **joomla** version.

```shell
nmap -p 80 --script http-enum TARGET
```

![](https://cdn.ziomsec.com/daily-bugle/5.webp)

I googled this version and found **sql** injection articles on it. I viewed the POC in **exploit-db** but didn't use it as the challenge asked us to use a python script instead.
- https://www.exploit-db.com/exploits/42033

I checked another article which had a link to a github repository that contained a python code.
- https://github.com/XiphosResearch/exploits/tree/master/Joomblah

I downloaded the script on my system and ran it against the target to get the user credentials.

```shell
python2 joomblah.py http://TARGET
```

![](https://cdn.ziomsec.com/daily-bugle/6.webp)

The script revealed the hash for `jonah`. To find it's type, I searched the **hashcat examples** page and found that it was a **bcrypt** hash.
- https://hashcat.net/wiki/doku.php?id=example_hashes

I then used **john** and found the password.

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt myhash
```

![](https://cdn.ziomsec.com/daily-bugle/7.webp)

I used the username revealed by the script and the cracked password to log into the **joomla administrative panel**

![](https://cdn.ziomsec.com/daily-bugle/8.webp)

I then looked for ways I could get an RCE from here and found a way on **hacktricks**.
- https://book.hacktricks.xyz/network-services/pentesting/pentesting-web/joomla

Hence, I first configured a **php** reverse shell payload on **revshells** with my IP and port.
- https://www.revshells.com

Then, as stated in the **hacktricks article**, I navigated to **Templates**. Here I found out that *`protostar`* was the default template being used on the web server.

![](https://cdn.ziomsec.com/daily-bugle/9.webp)

I then clicked on `Templates` and selected the `Protostar` template.

![](https://cdn.ziomsec.com/daily-bugle/10.webp)

Then, I selected the **index.php** code and pasted my reverse shell code and saved and closed it.

![](https://cdn.ziomsec.com/daily-bugle/11.webp)

Then I started an **nc** listener and visited the main page to get a reverse shell.

![](https://cdn.ziomsec.com/daily-bugle/12.webp)

I explored the directories that were present in `/var/www/html` and found a set of **mysql** credentials. I looked inside the **sql** server but found nothing useful. So, I just copied the password as it could be used in the future.

```shell
cat /var/www/html/configuration.php
```

![](https://cdn.ziomsec.com/daily-bugle/13.webp)

Since the user flag wasn't in the `/var/www` directory, I tried checking the `/home` directory. However, when I tried going inside **jjameson**, I was denied from accessing it.

I tried switching to this user with the password I had discovered earlier which luckily worked. 

```shell
su jjameson
```

![](https://cdn.ziomsec.com/daily-bugle/14.webp)

I then viewed inside the **jjameson** directory and found the user flag.

![](https://cdn.ziomsec.com/daily-bugle/15.webp)

## Privilege Escalation

I checked my **sudo** permissions by typing `sudo -l` and found that **jjameson** could execute `yum` as **sudo** without a password.

![](https://cdn.ziomsec.com/daily-bugle/16.webp)

I navigated to **gtfobins** and found a way to spawn an interactive shell as **root**.
- https://gtfobins.github.io/gtfobins/yum/

I simply copy pasted these commands on my terminal and spawned a **root** shell.

![](https://cdn.ziomsec.com/daily-bugle/17.webp)

![](https://cdn.ziomsec.com/daily-bugle/18.webp)

After becoming **root**, I captured the final flag from the `/root` directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/daily-bugle/19.webp)

## Closure

Here's a short summary of how I pwned **daily bugle**:
- I discovered an administrator login panel on the server from the **robots.txt** file revealed in the **nmap** scan.
- I used the **http-enum** script to find the **joomla** version.
- The version was vulnerable to **sql** injection. I used a python POC script from [github](https://github.com/stefanlucas/Exploit-Joomla/tree/master) to get the username and hash for the panel.
- I cracked the hash using **john** and logged into the application.
- I added a **php** reverse shell script in the default template used by the application and got a reverse shell.
- I found the sql credentials in the `/var/www/html/configurations.php` file.
- I managed to switch to `jjameson` using the password found above and captured the user flag from `jjameson`'s home directory.
- I exploited **yum**'s **sudo** privilege to get **root** access and captured the final flag from `/root`.

That's it from my side!
Until next time :)

---