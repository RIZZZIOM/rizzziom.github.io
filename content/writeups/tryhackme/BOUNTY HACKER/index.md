---
title: "Bounty Hacker"
date: 2025-06-20
draft: false
summary: "Writeup for Bounty Hacker CTF challenge on TryHackMe."
tags: ["linux", "ftp anonymous", "sudo", "bruteforce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/bountyhacker/cover.webp"
  caption: "Bounty Hacker TryHackMe Challenge"
  alt: "Bounty Hacker cover"
platform: "TryHackMe"
---

You were boasting on and on about your elite hacker skills in the bar and a few Bounty Hunters decided they'd take you up on claims! Prove your status is more than just a few glasses at the bar. I sense bell peppers & beef in your future!
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/room/cowboyhacker

## Reconnaissance

I performed an **nmap** scan on the target to find open ports and services running on it

```shell
nmap -sV -sC TARGET -T4 -oN bounty.nmap -Pn
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 80       | http        |
 
![](https://cdn.ziomsec.com/bountyhacker/1.webp)

> The nse scripts also found that the FTP server allowed *anonymous* access.

## Initial Foothold

I visited the website and found potential usernames. Besides that, there was nothing else. Even directory and file fuzzing yielded no results.

![](https://cdn.ziomsec.com/bountyhacker/2.webp)

I then moved onto FTP and logged in as an *anonymous* user. I then listed the contents and found 2 *txt* files.

```shell
ftp TARGET
```

![](https://cdn.ziomsec.com/bountyhacker/3.webp)

I downloaded both the files on my local system to view what's inside them.

![](https://cdn.ziomsec.com/bountyhacker/4.webp)

The *'task.txt'* file revealed 2 potential usernames:
- Vicious
- lin

The *locks.txt* file seemed like a wordlist.

![](https://cdn.ziomsec.com/bountyhacker/5.webp)

I then used **hydra** and found a valid **ssh** password from the *'locks.txt'* wordlist for the user *lin*.

```shell
hydra -l 'lin' -P locks.txt ssh://TARGET
```

![](https://cdn.ziomsec.com/bountyhacker/6.webp)

I accessed the target using **ssh** and captured the user flag from *lin*'s Desktop.

```shell
ssh lin@TARGET
cat user.txt
```

![](https://cdn.ziomsec.com/bountyhacker/7.webp)

## Privilege Escalation

Since I had the password, I looked at *lin*'s **sudo** privileges and found that I was allowed to execute **tar** as root.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/bountyhacker/8.webp)

I checked **GTFObins** to see how I could exploit this to escalate my privileges.
- https://gtfobins.github.io/gtfobins/tar/#sudo

I referred to the command in **GTFObins** to spawn a **bash** shell as *root*.

```shell
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

![](https://cdn.ziomsec.com/bountyhacker/9.webp)

Finally, I captured the root flag from */root* directory.

![](https://cdn.ziomsec.com/bountyhacker/10.webp)

That's it from my side!
Until next time :)

---
