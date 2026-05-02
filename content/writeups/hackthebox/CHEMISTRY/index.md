---
title: "Chemistry - HackTheBox Writeup"
date: 2025-01-17
draft: false
summary: "Writeup for Chemistry HackTheBox challenge."
tags: ["linux", "rce", "lfi", "web"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/chemistry/cover.webp"
  caption: "Chemistry HackTheBox Challenge"
  alt: "Chemistry cover"
platform: "HackTheBox"
author: "Moiz Bootwala"
---

**Chemistry** is a web‑focused Linux machine that centers on a web application and internal services. It requires careful enumeration and multi‑step escalation to capture user and root flags.
<!--more-->
To access the machine click on the link given below:
- https://www.hackthebox.com/machines/chemistry

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN chemistry.nmap
```

| **PORT** | **SERVICE** |
| -------- | ----------- |
| 22       | ssh         |
| 5000     | http        |

![nmap scan on chemistry to identify open ports and services](https://cdn.ziomsec.com/chemistry/1.webp)

## Foothold

The **nmap** scan revealed only 2 ports, hence I accessed the port running **http** service using **firefox**.

![accessing the web application running](https://cdn.ziomsec.com/chemistry/2.webp)

I registered using `test`:`test` 

![registering with test credentials](https://cdn.ziomsec.com/chemistry/3.webp)

The application allowed to upload a **CIF** file.

![file upload functionality](https://cdn.ziomsec.com/chemistry/4.webp)

Since the **nmap** scan revealed the backend to be running on **python**, I tried uploading a **python** script but failed.

![uploading a python script](https://cdn.ziomsec.com/chemistry/5.webp)

![error message on upload](https://cdn.ziomsec.com/chemistry/6.webp)

I then uploaded a **CIF** file to analyze the behavior using burp suite but found nothing interesting. The application also contained a dummy CIF file for us to download, so I downloaded it.

![](https://cdn.ziomsec.com/chemistry/7.webp)

Since I found nothing of interest, I googled vulnerabilities that could be exploited and found interesting articles.

![downloading example cif file](https://cdn.ziomsec.com/chemistry/8.webp)

I visited **revshells** and generated a simple **bash** payload.

![searching for cif rce](https://cdn.ziomsec.com/chemistry/9.webp)

I modified the **CIF** file and added the payload in it.

![configuring reverse shell with revshells](https://cdn.ziomsec.com/chemistry/10.webp)

Finally I uploaded the malicious CIF file.

![modifying test cif](https://cdn.ziomsec.com/chemistry/11.webp)

I started a reverse shell listener using **netcat** and when I executed the CIF file, I got a reverse shell. I spawned a **tty** shell and exported my terminal for better shell functionality.

```shell
$ rlwrap nc -lvp 80
$ python3 -c "import pty;pty.spawn('/bin/bash')"
$ export TERM=xterm
```

![reverse shell from the server](https://cdn.ziomsec.com/chemistry/12.webp)

I discovered the existence of an sqlite file along with a password.

```shell
cat app.py
```

![found hardcoded credentials](https://cdn.ziomsec.com/chemistry/13.webp)

![found potential users](https://cdn.ziomsec.com/chemistry/14.webp)

Then I tried looking for the flag. I found it in the home directory of another user called **rosa**. However, since I did not have enough permissions, I could not access it.

![tried accessing the user flag](https://cdn.ziomsec.com/chemistry/16.webp)

### Shell As `rosa`

I then viewed the sqlite database file and found the md5 hashes of different users including **rosa**.

```shell
sqlite3 instance/database.db
```

![connecting to sqlite instance and viewing password hashes](https://cdn.ziomsec.com/chemistry/15.webp)

I visited **crackstation** and cracked the password hash.

![cracking rosa hash with crackstation](https://cdn.ziomsec.com/chemistry/17.webp)

Finally, I used the credentials to log in as **rosa** and captured the user flag from the home directory.

```shell
ssh rosa@TARGET
# enter password
```

![ssh as rosa](https://cdn.ziomsec.com/chemistry/18.webp)

![capturing user flag](https://cdn.ziomsec.com/chemistry/19.webp)

## Privilege Escalation

After capturing the user flag, I downloaded and ran **linux smart enumeration** script to look for ways to escalate privilege, however the script found nothing interesting. I then looked at listening ports and found port **8080** bounded to localhost on listen mode.

```shell
netstat -antp
```

![listing listening ports](https://cdn.ziomsec.com/chemistry/20.webp)

To access the internally bounded port, I performed port forwarding, binding port 1234 to the localhost port 8080 of the target.

```shell
ssh -L 1234:127.0.0.1:8080 rosa@TARGET
```

![local port forwarding to access internal service](https://cdn.ziomsec.com/chemistry/21.webp)

After that, I was able to access the service running on port 8080.

![accessing the internal service](https://cdn.ziomsec.com/chemistry/22.webp)

I performed an **nmap** scan on the service and found the service version.

```shell
nmap -A -p 1234 127.0.0.1
```

![performing nmap scan on the internal service](https://cdn.ziomsec.com/chemistry/23.webp)

further footprinting revealed a **path traversal** vulnerability associated with this particular service version.

![finding path traversal vulnerability](https://cdn.ziomsec.com/chemistry/24.webp)

Since the vulnerability was path traversal, I looked for directories on the target using **ffuf**.

```shell
ffuf -u http://127.0.0.1:1234/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 404
```

![performing fuzz scan on internal service](https://cdn.ziomsec.com/chemistry/25.webp)

I then found a PoC of the CVE on **github** and got a reference on how the vulnerability could be exploited. 
- https://github.com/jhonnybonny/CVE-2024-23334

I then used **curl** and tried accessing the **/etc/passwd** file by exploiting the **path traversal** vulnerability.

```shell
curl --path-as-is 'http://127.0.0.1:1234/assets/../../../../etc/passwd'
```

![accessing /etc/passwd](https://cdn.ziomsec.com/chemistry/26.webp)

Since I was able to confirm the vulnerability, I exploited it to access the root flag from the **/root** directory.

```shell
curl --path-as-is 'http://127.0.0.1:1234/assets/../../../../root/root.txt'
```

![accessing the root flag](https://cdn.ziomsec.com/chemistry/27.webp)

## Closure

Here's a short summary of how I pwned **chemistry**:
- Ran a full port scan and found SSH and an HTTP app.
- Registered on the web app and discovered a CIF file upload feature.
- Uploaded a crafted CIF containing a bash reverse-shell payload and gained a remote shell.
- Found an SQLite database with MD5 password hashes, cracked a hash to obtain rosa’s credentials, and logged in as rosa.
- Enumerated and found a service bound to localhost, then used SSH port forwarding to access it locally.
- Identified a path‑traversal vulnerability in that service and used it to read the root flag.

Happy Hacking !

---
