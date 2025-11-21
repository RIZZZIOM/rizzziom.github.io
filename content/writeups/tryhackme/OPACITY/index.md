---
title: "Opacity"
date: 2024-04-05
draft: false
summary: "Writeup for Opacity CTF challenge on TryHackMe."
tags: ["linux", "file upload", "misconfigured service"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/opacity/cover.webp"
  caption: "Opacity TryHackMe Challenge"
  alt: "Opacity cover"
platform: "TryHackMe"
---

Opacity is a Boot2Root made for pentesters and cybersecurity enthusiasts.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/opacity

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN opacity.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/opacity/1.webp)

## Initial Foothold

I accessed the web application running on port 80 and found a login panel.

![](https://cdn.ziomsec.com/opacity/2.webp)

I fuzzed hidden directories and found an interesting directory called `cloud`.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/opacity/3.webp)

I accessed the `cloud` endpoint and found a file upload functionality.

![](https://cdn.ziomsec.com/opacity/4.webp)

I then created a reverse shell php payload and served it using python's http server on my local system.

![](https://cdn.ziomsec.com/opacity/5.webp)

I tried uploading it directly, however the application had some sort of validation mechanism in place that only allowed image files. So I used `#` to add `.jpg` extension after my php payload. Fragment identifiers (everything after `#`) **are not sent to the server** by the browser. So when your request is sent, it removes `.jpg` and sends the php reverse shell. 

I was able to bypass the security mechanism and upload the payload.

![](https://cdn.ziomsec.com/opacity/6.webp)

The payload got executed after being uploaded and I got a reverse shell on my **netcat** listener.

```shell
nc -lvnp 1234
```

![](https://cdn.ziomsec.com/opacity/7.webp)

I then tried accessing the user flag from sysadmin's home directory but didn't have the required privilege.

![](https://cdn.ziomsec.com/opacity/8.webp)

## Privilege Escalation

### Shell As Opacity.

Since I did not have permissions to read the user flag, I would have to escalate my privileges. Hence I read the source code of the login page and found the login credentials.

![](https://cdn.ziomsec.com/opacity/9.webp)

I logged in using the credentials but found nothing interesting on the application. I then explored other directories and found a keepass password database file inside the `/opt` directory.

![](https://cdn.ziomsec.com/opacity/10.webp)

To know more about keepass, I referred to the below artciles:
- https://fileinfo.com/extension/kdbx
- https://gist.github.com/lgg/e6ccc6e212d18dd2ecd8a8c116fb1e45

I downloaded the file on my local system and downloaded keepass on my windows system. However, when I tried opening the file, it prompted for a password.

To crack the password, I converted it into crackable format using `keepass2john`.

```
keepass2john dataset.kdbx > dataset.hash
```

![](https://cdn.ziomsec.com/opacity/11.webp)

I then cracked its password using **john**.

```shell
john --wordlists=/usr/share/wordlists/rockyou.txt dataset.hash
```

![](https://cdn.ziomsec.com/opacity/12.webp)

I used the password to view the contents of the database and found user credentials.

![](https://cdn.ziomsec.com/opacity/13.webp)

![](https://cdn.ziomsec.com/opacity/14.webp)

I then logged in using these credentials and captured the user flag.

```shell
ssh sysadmin@TARGET
```

![](https://cdn.ziomsec.com/opacity/15.webp)

### Shell As Root

I downloaded **linux smart enumeration** script on the target and ran it to find privileges escalation vectors.
- https://github.com/diego-treitos/linux-smart-enumeration

![](https://cdn.ziomsec.com/opacity/16.webp)

We had read access to a backup file. I also found a script in sysadmin's home directory that used the backup.zip file to save a backup of the scripts folder.

![](https://cdn.ziomsec.com/opacity/17.webp)

It used a file called `backup.inc.php` So I replaced that file with my php reverse shell.

![](https://cdn.ziomsec.com/opacity/18.webp)

I started a netcat listener

```shell
rlwrap nc -lnvp 1234
```

After transferring the reverse shell with the name as `backup.inc.php`, I got a reverse shell.

![](https://cdn.ziomsec.com/opacity/19.webp)

After gaining root access, I captured proof.txt from `/root`.

```shell
cat /root/proof.txt
```

![](https://cdn.ziomsec.com/opacity/20.webp)

That's it from my side, until next time !

---
