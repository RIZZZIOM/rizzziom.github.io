---
title: "Lianyu"
date: 2025-10-13
draft: false
summary: "Writeup for Lianyu CTF challenge on TryHackMe."
tags: ["linux", "sudo", "steganography"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lianyu/cover.webp"
  caption: "Lianyu TryHackMe Challenge"
  alt: "Lianyu cover"
platform: "TryHackMe"
---

A beginner level security challenge
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/lianyu

## Reconnaissance

I scanned the target using **nmap** to find its open ports, services etc.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN lianyu.nmap
```

| **Port** | **Target** |
| -------- | ---------- |
| 21       | ftp        |
| 22       | ssh        |
| 80       | http       |
| 111      | rpc        |
| 33628    | rpc        |

![](https://cdn.ziomsec.com/lianyu/1.webp)

## Initial Foothold

Since the target was running a web application, I fuzzed it for hidden directories and found an interesting endpoint.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![](https://cdn.ziomsec.com/lianyu/2.webp)

I visited the endpoint and felt that the information on the page was incomplete.

![](https://cdn.ziomsec.com/lianyu/3.webp)

So, I viewed the source code and found a potential username/password.

![](https://cdn.ziomsec.com/lianyu/4.webp)

I then fuzzed for hidden directories inside the newly discovered endpoint and found another endpoint.

```shell
ffuf -u http://TARGET/island/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![](https://cdn.ziomsec.com/lianyu/5.webp)

Upon visiting the page, I viewed the source code and found an interesting comment left by the developer.

![](https://cdn.ziomsec.com/lianyu/6.webp)

The comment said we could avail our `.ticket`... Maybe, there could be a file or directory on this endpoint with the `.ticket` extension.

![](https://cdn.ziomsec.com/lianyu/7.webp)

So, I fuzzed for `.ticket` endpoints inside the hidden directory.

```shell
ffuf -u http://TARGET/island/2100/FUZZ.ticket -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fc 403 -fs 292
```

![](https://cdn.ziomsec.com/lianyu/8.webp)

I accessed the newly discovered endpoint and found an encoded piece of string.

![](https://cdn.ziomsec.com/lianyu/9.webp)

I tried various methods to decode the string and successfully decoded it when I used the *From Base58* decoder from **cyberchef** : `!#th3h00d`
- https://gchq.github.io/CyberChef

This looked like a password so I checked if this could be used with the username that I had found earlier on the other services running on the target i.e **ssh** or **ftp**. The credentials worked with **ftp*.

```shell
hydra -l 'vigilante' -p '!#th3h00d' ftp://TARGET
```

![](https://cdn.ziomsec.com/lianyu/10.webp)

I connected to the ftp server and listed the available files.

```shell
ftp TARGET
```

It contained some images so I downloaded them onto my local system.

```shell
get aa.jpg
get Leave_me_alone.png
get Queen's_Gambit.png
```

![](https://cdn.ziomsec.com/lianyu/11.webp)

The `leave_me_alone.png` file seemed to have some problem.

![](https://cdn.ziomsec.com/lianyu/12.webp)

The Exif data confirmed that there was a file format error.

```shell
exiftool Leave_me_alone.png
```

![](https://cdn.ziomsec.com/lianyu/13.webp)

I uploaded the image in a hex editor and found that the magic header numbers were incorrect. The headers for png should be: `89 50 4E 47 0D 0A 1A 0A`

![](https://cdn.ziomsec.com/lianyu/14.webp)

I fixed the headers and downloaded the image.

![](https://cdn.ziomsec.com/lianyu/15.webp)

However, the image contained nothing interesting.

![](https://cdn.ziomsec.com/lianyu/16.webp)

I then tried extracting data from `aa.jpg` and found out it contained a zip file. Unzipping it revealed 2 files.

```shell
stegseek aa.jpg
unzip aa.jpg.out
```

![](https://cdn.ziomsec.com/lianyu/17.webp)

One file contained a password while the other had some kind of a note.

![](https://cdn.ziomsec.com/lianyu/18.webp)

At this point, I tried tried fuzzing spraying this password with the potential usernames that I had found so far, however, nothing worked. I went back to the **ftp** server and listed the hidden files as well to find an interesting hidden file called *`.other_user`*.

![](https://cdn.ziomsec.com/lianyu/19.webp)

I downloaded the file on my local system.

```shell
get .other_user
```

![](https://cdn.ziomsec.com/lianyu/20.webp)

The file contained a bunch of potential usernames.

![](https://cdn.ziomsec.com/lianyu/21.webp)

I tried the password with these usernames aswell and found a valid ssh credential.

```shell
hydra -l 'slade' -p 'M3tahuman' ssh://TARGET
```

![](https://cdn.ziomsec.com/lianyu/22.webp)

I connected to the machine using the discovered credentials through **ssh**.

```shell
ssh slade@TARGET
```

![](https://cdn.ziomsec.com/lianyu/23.webp)

Finally, I captured the user flag from my home directory.

```shell
cat user.txt
```

![](https://cdn.ziomsec.com/lianyu/24.webp)

## Privilege Escalation

I then listed my **sudo** privileges and found that I was allowed to run the **pkexec** binary as root without password.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/lianyu/25.webp)

I referred to  **gtfobins** and found a way to exploit this to get root access.
- https://gtfobins.github.io/gtfobins/pkexec/

```shell
sudo pkexec /bin/bash
```

![](https://cdn.ziomsec.com/lianyu/26.webp)

I then captured the root flag from the root user's home directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/lianyu/27.webp)

That's it from my side!
Until next time :)

---
