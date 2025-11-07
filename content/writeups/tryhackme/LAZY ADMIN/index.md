---
title: "Lazy Admin"
date: 2025-05-25
draft: false
summary: "Writeup for Lazy Admin CTF challenge on TryHackMe."
tags: ["linux", "cms", "sudo", "file upload"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lazyadmin/cover.webp"
  caption: "Lazy Admin TryHackMe Challenge"
  alt: "Lazy Admin cover"
platform: "TryHackMe"
---

Easy linux machine to practice your skills
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/lazyadmin

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN lazyadmin.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/lazyadmin/1.webp)

## Initial Foothold

The **nmap** scan revealed an http server, so I accessed the web page using my browser.

![](https://cdn.ziomsec.com/lazyadmin/2.webp)

I then looked for hidden directories using **ffuf**.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/lazyadmin/3.webp)

I visited the `/content/` directory and found some text related to `SweetRice CMS`. So, I dug deeper to find endpoints within it.

```shell
ffuf -u http://TARGET/content/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/lazyadmin/4.webp)

![](https://cdn.ziomsec.com/lazyadmin/5.webp)

I discovered a login panel and a couple of directories.

![](https://cdn.ziomsec.com/lazyadmin/6.webp)

The `/inc/` folder seemed to have the backend codes.

![](https://cdn.ziomsec.com/lazyadmin/7.webp)

I found the service version at `/content/inc/latest.txt`.

![](https://cdn.ziomsec.com/lazyadmin/8.webp)

The `/inc` folder contained a directory called *mysql_backup* which could be of interest. I inspected it and found an SQL file. To extract its contents, I downloaded it.

![](https://cdn.ziomsec.com/lazyadmin/9.webp)

A simple `cat` command revealed some credentials that could be used to log into the CMS or through **ssh**.

```shell
cat mysql_bakup_20191129023059-1.5.1.sql | grep pass
```

![](https://cdn.ziomsec.com/lazyadmin/10.webp)

I cracked the hash on **crackstation** and found the password : `Password123`
- https://crackstation.net/

I then logged into the CMS using the credentials.

| username | password    |
| -------- | ----------- |
| manager  | Password123 |

The dashboard contained the version of the CMS that was being used. So I searched for exploits available using **searchsploit**.

![](https://cdn.ziomsec.com/lazyadmin/11.webp)

```shell
searchsploit 'sweetrice 1.5.1'
```

![](https://cdn.ziomsec.com/lazyadmin/12.webp)

### Abusing File Upload

There seemed to be a file upload vulnerability, so I downloaded **pentest monkey's** php reverse shell code and tried uploading it in the **media center** page:
- https://github.com/pentestmonkey/php-reverse-shell

![](https://cdn.ziomsec.com/lazyadmin/13.webp)

However, due to security reasons, I wasn't able to upload it on the target. I then tried using an alternate extension like **.phtml** and successfully bypassed the security.

![](https://cdn.ziomsec.com/lazyadmin/14.webp)

I then started a **netcat** listener and executed the payload to get a reverse shell.

![](https://cdn.ziomsec.com/lazyadmin/15.webp)

I then captured the user flag from *itguy*'s home directory.

```shell
cat /home/itguy/user.txt
```

![](https://cdn.ziomsec.com/lazyadmin/16.webp)

## Privilege Escalation

I also found a **mysql** login credential so I connected to the server using it.

![](https://cdn.ziomsec.com/lazyadmin/17.webp)

```shell
mysql -u rice -p
```

![](https://cdn.ziomsec.com/lazyadmin/18.webp)

However, I found nothing interesting. Next I looked for my **sudo** privileges and found that I was allowed to run a perl script.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/lazyadmin/19.webp)

I viewed the contents of the perl script and found that it executed a bash script that I was allowed to modify. To escalate my privilege, I added a command that would spawn a bash shell in privileged mode.

```shell
cat backup.pl
cat /etc/copy.sh
echo '/bin/bash -p' >> /etc/copy.sh
```

![](https://cdn.ziomsec.com/lazyadmin/20.webp)

![](https://cdn.ziomsec.com/lazyadmin/21.webp)

Finally, I executed the perl script with **sudo**

```shell
sudo /usr/bin/perl /home/itguy/backup.pl
```

![](https://cdn.ziomsec.com/lazyadmin/22.webp)

After getting root access, I captured the final flag.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/lazyadmin/23.webp)

That's it from my side!
Happy hacking :)

---
