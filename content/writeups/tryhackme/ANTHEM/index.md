---
title: "Anthem"
date: 2025-02-14
draft: false
summary: "Writeup for Anthem CTF challenge on TryHackMe."
tags: ["windows", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/anthem/cover.webp"
  caption: "Anthem TryHackMe Challenge"
  alt: "Anthem cover"
platform: "TryHackMe"
---

Exploit a Windows machine in this beginner level challenge.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/anthem

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -p- -A TARGET -oN anthem.nmap --min-rate 10000 -Pn
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |
| 3389     | rdp         |

![](https://cdn.ziomsec.com/anthem/1.webp)

## Initial Foothold

The **nmap** scan revealed an **http** server running on the target. So I accessed it from my browser.

![](https://cdn.ziomsec.com/anthem/2.webp)

I used **ffuf** to bruteforce hidden files on the web app.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/anthem/3.webp)

The *robots.txt* file usually contained sensitive endpoints. So I accessed it and found a string. I saved the string as it could be used in the future.

![](https://cdn.ziomsec.com/anthem/4.webp)

I continued analyzing the application and found an email address.

![](https://cdn.ziomsec.com/anthem/5.webp)

![](https://cdn.ziomsec.com/anthem/6.webp)

When I tried accessing the page of the author `jane-doe`, I received an error. So I accessed the *authors* page and discovered the first flag.

![](https://cdn.ziomsec.com/anthem/7.webp)

![](https://cdn.ziomsec.com/anthem/8.webp)

I reviewed the source code of my home page and found another flag.

![](https://cdn.ziomsec.com/anthem/9.webp)

I analyzed the remaining endpoints uncovered by robots.txt and the `fuff` scan and found a login panel at the `umbraco` endpoint.

![](https://cdn.ziomsec.com/anthem/10.webp)

I was able to log in using the email id I found on the web page and using the string from *robots.txt* as password and got access to a CMS admin panel.

![](https://cdn.ziomsec.com/anthem/11.webp)

I viewed the source code and got another flag.

![](https://cdn.ziomsec.com/anthem/12.webp)

I viewed the source code of other pages from here and found another flag in `/archive/a-cheers-to-our-it-department/`

![](https://cdn.ziomsec.com/anthem/13.webp)

I checked if the credentials I used on the web app could be used to get **rdp** access using **nxc**.

```shell
nxc rdp TARGET -u SG -p 'UmbracoIsTheBest!'
```

![](https://cdn.ziomsec.com/anthem/14.webp)

Since I had the valid credentials, I got **rdp** access on the target.

```shell
remmina
```

I then captured the user flag from the user's desktop.

![](https://cdn.ziomsec.com/anthem/15.webp)

## Privilege Escalation

I looked at the folders and files present (including hidden files) and found a backup folder inside the C drive. This folder contained a file called *restore.txt* that I could not access.

![](https://cdn.ziomsec.com/anthem/16.webp)

Hence, I modified the file's properties to allow the machine account to read the file.

![](https://cdn.ziomsec.com/anthem/17.webp)

![](https://cdn.ziomsec.com/anthem/18.webp)

![](https://cdn.ziomsec.com/anthem/19.webp)

Finally I read the file and found a string. I was then able to log in as `Administrator` using this as a password.

![](https://cdn.ziomsec.com/anthem/20.webp)

I then captured the final flag from desktop.

![](https://cdn.ziomsec.com/anthem/21.webp)

That's it from my side !
Happy hacking :)

---
