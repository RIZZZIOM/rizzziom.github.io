---
title: "MHZ_C1F"
date: 2024-10-17
draft: false
summary: "Writeup for MHZ_C1F CTF challenge on VulnHub."
tags: ["linux", "sudo", "steganography"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/mhzctf/cover.webp"
  caption: "MHZ_C1F VulnHub Challenge"
  alt: "MHZ_C1F cover"
platform: "VulnHub"
---

**MHZC1F** is a boot2root challenge comprising of 2 flags. We are required to capture them both.
<!--more-->
To download **MHZ_C1F** - click on the link given below:
- https://www.vulnhub.com/entry/mhz_cxf-c1f,471/

## Recon

I started of by performing an **nmap** aggressive scan on the target to identify open ports and the services running on them.

```shell
nmap -A -p- --min-rate 10000 -oN mhz.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/mhzctf/1.webp)

## Foothold

I visited the site running on port 80 and found a default landing page.

![](https://cdn.ziomsec.com/mhzctf/2.webp)

I performed a directory and file fuzz using **ffuf** and found a file **notes.txt**.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/mhzctf/3.webp)

I accessed the path and found a message hinting towards another directories. The contents of `remb.txt` looked like credentials, and the second file didn't seem to exist on the website.

```shell
curl -s http://TARGET/notes.txt
curl -s http://TARGET/remb.txt
curl -s http://TARGET/remb2.txt
```

![](https://cdn.ziomsec.com/mhzctf/4.webp)

![](https://cdn.ziomsec.com/mhzctf/5.webp)

I saved the credentials and tried using it to log in through **ssh**.

![](https://cdn.ziomsec.com/mhzctf/6.webp)

I got a generic shell, so I spawned a pty shell using **python**.

```shell
python3 -c 'import pty'
```

![](https://cdn.ziomsec.com/mhzctf/7.webp)

I found the first flag in the user's home directory.

![](https://cdn.ziomsec.com/mhzctf/8.webp)

## Privilege Escalation

The home directory contained a folder for another user called `mhz_c1f` which had a directory of painting images.

```shell
cd mhz_c1f/
ls -la Paintings/
```

![](https://cdn.ziomsec.com/mhzctf/9.webp)

This looked interesting so I copied those paintings onto my system using **scp** as ssh was enabled on the target.

```shell
scp first_stage@TARGET:/home/mhz_c1f/Paintings/* paintings
```

![](https://cdn.ziomsec.com/mhzctf/10.webp)

I used **binwalk** to find information about the images.

```shell
binwalk 'FILENAME'
```

![](https://cdn.ziomsec.com/mhzctf/11.webp)

I then tried extracting data from each image with a blank password. I failed on the first 3 images but, the information from the last image was successfully extracted.

```shell
steghide --extract -sf 'FILENAME'
```

![](https://cdn.ziomsec.com/mhzctf/12.webp)

I read the file and found another set of credentials.

![](https://cdn.ziomsec.com/mhzctf/13.webp)

I tried to use this to log in through **ssh** but it didn't work.

```shell
ssh mhz_c1f@TARGET
```

![](https://cdn.ziomsec.com/mhzctf/14.webp)

I then tried switching my user from the shell I already had and was able to successfully do so.

```shell
su mhz_c1f
```

![](https://cdn.ziomsec.com/mhzctf/15.webp)

I then ran the **linux-smart-enumeration** script and found a very interesting configuration.

![](https://cdn.ziomsec.com/mhzctf/16.webp)
![](https://cdn.ziomsec.com/mhzctf/17.webp)

The user was allowed to run all commands as sudo without any password. So I cross checked this manually and used it to spawn a bash shell as root.

```shell
sudo -l
sudo /bin/bash
```

![](https://cdn.ziomsec.com/mhzctf/18.webp)

Finally, I captured the root flag from the root user's home directory.

![](https://cdn.ziomsec.com/mhzctf/19.webp)

## Closure

Here's a summary of how I pwned the machine:
- I found a set of credentials by web fuzzing.
- I was able to log in through ssh using those credentials.
- The home directory had another user who had a couple of images.
- I transferred those images onto my system and performed steganography to reveal another set of credentials.
- I used those to switch my user.
- I checked the sudo privileges of this user and found the user could run sudo commands without a password.
- I leveraged this vulnerability to spawn a bash shell as root and captured the final flag from the root user's home directory.

That's it from my side:) 
Until next time!

---
