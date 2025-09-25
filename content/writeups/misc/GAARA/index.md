---
title: "Gaara"
date: 2024-11-01
draft: false
summary: "Writeup for the Gaara proving grounds CTF challenge."
tags: ["linux", "hardcoded creds", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/gaara/cover.webp"
  caption: "Gaara Proving Grounds Challenge"
  alt: "Gaara cover"
platform: "Misc"
---

**Gaara** is an easy CTF challenge on Offsec'c Proving Grounds and requires us to capture 2 flags hidden in the system.
<!--more-->
To access the machine, click on the link given below:
- https://portal.offsec.com/labs/play

## Recon

I performed an **nmap** aggressive scan on the target to identify open ports and the services running on them along with some additional information.

```shell
nmap -A -p- TARGET -oN gaara.nmap --min-rate 10000
```

| **Port** | **Service**            |
| -------- | ---------------------- |
| 22       | ssh                    |
| 80       | http                   |

![](https://cdn.ziomsec.com/gaara/1.webp)

## Initial Foothold

The **nmap** scan discovered an **http** server running on the target. So I accessed the site on browser.

![](https://cdn.ziomsec.com/gaara/2.webp)

The website had nothing interesting so I used **ffuf** to perform web directory fuzzing. Through this, I discovered a new directory.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/gaara/3.webp)

Upon accessing the directory, I found 3 new paths.

![](https://cdn.ziomsec.com/gaara/4.webp)

I accessed them one after the other but they all had the same extract. Upon closer inspection, I found that `/iamGaara` had an encoded string.

![](https://cdn.ziomsec.com/gaara/5.webp)

![](https://cdn.ziomsec.com/gaara/6.webp)

![](https://cdn.ziomsec.com/gaara/7.webp)

To decode this, I visited **cyberchef** and tried multiple encoding formats, out of which **base58** worked. 

![](https://cdn.ziomsec.com/gaara/8.webp)

This looked like a pair of credentials so I tried logging in through **ssh**. However, it turned out to be a rabbit hole. The password was incorrect.

![](https://cdn.ziomsec.com/gaara/9.webp)

Since I had no other leads, I tried brute-forcing the password of  *gaara* using **hydra** and **rockyou.txt** wordlist.

```shell
hydra -l 'gaara' -P /usr/share/wordlists/rockyou.txt ssh://TARGET
```

![](https://cdn.ziomsec.com/gaara/10.webp)

After finding the valid password, I logged in through **ssh** and found the flag in my home directory.

```shell
ssh gaara@TARGET
```

![](https://cdn.ziomsec.com/gaara/11.webp)

## Privilege Escalation

After getting initial access, I downloaded **linux smart enumeration** script on the target to look for misconfigurations that could lead to privilege escalation.

![](https://cdn.ziomsec.com/gaara/12.webp)

I ran the script and found an interesting set of binaries with **setuid** bit. I also found that I could write into the `/usr/local/games` directory.

![](https://cdn.ziomsec.com/gaara/13.webp)

![](https://cdn.ziomsec.com/gaara/14.webp)

![](https://cdn.ziomsec.com/gaara/15.webp)

I started off with the **setuid** bit binaries and searched **gtfobins** for ways to exploit them. I found a way to escalate privilege with **gdb** so I repeated the steps mentioned on the site.

```shell
gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit
```

![](https://cdn.ziomsec.com/gaara/16.webp)

![](https://cdn.ziomsec.com/gaara/17.webp)

After getting **root** access, I captured the final flag from the `/root` directory.

![](https://cdn.ziomsec.com/gaara/18.webp)

## Closure

Here's a summary of how I pwned the machine:
- I performed a directory brute-force attack to discover hidden directories.
- This directory revealed more paths that I could analyze.
- One of the page contained a base58 encoded string which contained a username and password.
- After failing to log in with the given password, I brute-forced the correct password using **hydra**.
- **hydra** identified the valid set of credentials and I used them to log in as **gaara**.
- I then captured the first flag in my home directory.
- I discovered uncommon setuid bits on 2 binaries and visited **gtfobins** to look for ways to exploit them,
- On **gtfobins**, I found a way to escalate my privilege using **gdb**
- I used the provided method to become a **root** user and captured the final flag from the `/root` directory.

That's it from my side!
Until next time :)

---
