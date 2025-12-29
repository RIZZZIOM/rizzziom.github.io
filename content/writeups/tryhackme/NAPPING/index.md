---
title: "Napping"
date: 2025-12-29
draft: false
summary: "Writeup for Napping CTF challenge on TryHackMe."
tags: ["linux", "phishing", "sudo", "cron"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/napping/cover.webp"
  caption: "Napping TryHackMe Challenge"
  alt: "Napping cover"
platform: "TryHackMe"
---

Even Admins can fall asleep on the job
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/nappingis1337

## Reconnaissance

I performed an **nmap** aggressive scan on the target to identify open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN napping.nmap -Pn
```

| **PORT** | **SERVICE** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/napping/1.webp)

## Initial Foothold

I accessed the web application through my browser and found a login page.

![](https://cdn.ziomsec.com/napping/2.webp)

Since I did not have the credentials to log in, I signed up from the Sign up page

![](https://cdn.ziomsec.com/napping/3.webp)

I then performed a **ffuf** scan and found the `admin` directory. However, I was forbidden from accessing the directory. I also discovered the admin login page at `/admin/login.php`.

```shell
ffuf -u http://TARGET/FUZZ/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 404
```

![](https://cdn.ziomsec.com/napping/4.webp)

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 404
```

![](https://cdn.ziomsec.com/napping/5.webp)

I then logged into the application with the credentials I had generated and found an interesting functionality to provide a url. To verify if a request was actually being made, I started a netcat listener and added the url on the application.

![](https://cdn.ziomsec.com/napping/6.webp)

![](https://cdn.ziomsec.com/napping/7.webp)

This confirmed that "admin" was visiting the URLs that were being sent. I inspected the source code and discovered a misconfiguration that could be used to perform tab nabbing.

![](https://cdn.ziomsec.com/napping/8.webp)

The link had the target attribute set to `_blank` and did not have attributes like `noopener` and `noreferrer`. So, we modify the the "opener"  that becomes inactive to steal credentials.

> **Tab nabbing** is a phishing-style attack that exploits how users interact with **inactive browser tabs**. We take advantage of a tab the user opened earlier (usually via a link that opens in a new tab) and change its content while the user is away, so when the user comes back, they believe they are still on a trusted site. **[Learn more](https://owasp.org/www-community/attacks/Reverse_Tabnabbing)**

Here's how I exploited this to get the admin credentials:
```
ADMIN OPENS ANY LINK SENT TO HIM → I CREATE 2 PAGES → ADMIN VISIT REDIRECTOR & FALLS FOR TRAP
									↓                ↓
								LOGIN.HTML       OPENME.HTML
					(replica of admin login)   (replaces inactive window to login replica)
```

I created the `openme.html` file to change the admin's inactive page into a replica of the admin login panel.

![](https://cdn.ziomsec.com/napping/9.webp)

I then copied the source code on `/admin/login.php` into `login.html`

![](https://cdn.ziomsec.com/napping/10.webp)

Finally, I started a web server to host both the files and wireshark to analyze the requests made

```shell
# start server on all interfaces on port 80
sudo php -S 0.0.0.0:80
```

![](https://cdn.ziomsec.com/napping/11.webp)

![](https://cdn.ziomsec.com/napping/12.webp)

After capturing the password, I logged into the target as *daniel*

```shell
ssh daniel@TARGET
```

![](https://cdn.ziomsec.com/napping/13.webp)

## Privilege Escalation

Executing the `id` command revealed I was part of a group called *administrators*.

### Shell As Adrian

I found the user flag in *adrean*'s directory but couldn't open it due to lack of permissions. However, there was a python script that was accessible to administrators group. I read the script and found a health check mechanism. I also found a file called *site_status.txt* which revealed the script ran every minute.

![](https://cdn.ziomsec.com/napping/14.webp)

I also found the sql credentials in `/var/www/html/config.php` but didn't find anything useful in the database.

![](https://cdn.ziomsec.com/napping/15.webp)

I then modified the `query.py` script to get a reverse shell on my netcat listener.

![](https://cdn.ziomsec.com/napping/16.webp)

I started a netcat listener and got a reverse shell as *adrean*.

![](https://cdn.ziomsec.com/napping/17.webp)

Finally, I captured the user flag.

![](https://cdn.ziomsec.com/napping/18.webp)

### Shell As Root

I then viewed my **sudo** privs and found I could run `vim` as root.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/napping/19.webp)

I referred to **[GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo)** and used the documented method to elevate my privileges to root.

```
sudo vim -c ':!/bin/bash'
```


![](https://cdn.ziomsec.com/napping/20.webp)

## Closure

Here's a short summary of how I pwned **Napping**:
- I discovered a web application running on the target where admin visited links submitted through a portal.
- The mechanism used to view the links was vulnerable to tab nabbing.
- I phished the admin user by replacing the inactive web app tab with a custom admin login page and captured the password that was entered.
- I used the credentials to log into the target as daniel and found that I was part of the *administrators* group.
- Members of *administrators* group had write access over a python script that ran every minute as user *adrian*.
- I added a python reverse shell exploit to the script and got a shell as *adrian*.
- *adrian* had privilege to run `vim` as **sudo**
- I exploited this to gain root access.

That concludes my writeup for **Napping**. Until next time :)

---