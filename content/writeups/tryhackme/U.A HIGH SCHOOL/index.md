---
title: "UA High School"
date: 2024-08-30
draft: false
summary: "Writeup for UA High School CTF challenge on TryHackMe."
tags: ["linux", "sudo", "steganography"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/ua-high-school/cover.webp"
  caption: "UA High School TryHackMe Challenge"
  alt: "UA High School cover"
platform: "TryHackMe"
---

Welcome to the web application of U.A., the Superhero Academy.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/yueiua

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET -oN ua.nmap --min-rate 10000
```

![](https://cdn.ziomsec.com/ua-high-school/1.webp)

## Foothold

The **nmap** scan revealed a web application running so I accessed it through my browser.

![](https://cdn.ziomsec.com/ua-high-school/2.webp)

I then used **ffuf** to find hidden directories on the web app.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 302,301
```

![](https://cdn.ziomsec.com/ua-high-school/3.webp)

I accessed the newly discovered directory but found nothing so I used ffuf to find hidden files.

```shell
ffuf -u http://TARGET/assets/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
```

![](https://cdn.ziomsec.com/ua-high-school/4.webp)

I accessed the files and found nothing at first. However, when I tried passing command through common variables on *index.php*, I received a url base64 encoded response.

![](https://cdn.ziomsec.com/ua-high-school/5.webp)

![](https://cdn.ziomsec.com/ua-high-school/6.webp)

Hence, I was able to execute os commands on the target. I viewed the source code of *index.php* using this.

![](https://cdn.ziomsec.com/ua-high-school/7.webp)

![](https://cdn.ziomsec.com/ua-high-school/8.webp)

I then checked if the machine had **netcat** so that I could try and initiate a reverse shell connection and after confirmation   visited **revshells** and copied an **nc mkfifo** command to get a reverse shell. Upon execution, I received a shell on my **netcat** listener.
- https://www.revshells.com

![](https://cdn.ziomsec.com/ua-high-school/9.webp)

![](https://cdn.ziomsec.com/ua-high-school/10.webp)

After spawning a pty shell, I found a passphrase that was base64 encoded.

```shell
cat /var/www/Hidden_Content/passphrase.txt
```

![](https://cdn.ziomsec.com/ua-high-school/11.webp)

Decoding it revealed a password.

![](https://cdn.ziomsec.com/ua-high-school/12.webp)

I found the user "`deku`" from the */home* directory and tried switching to it using the password but failed.

I then looked deeper and found some images inside the *assets* directory.

![](https://cdn.ziomsec.com/ua-high-school/13.webp)

I downloaded the images on my local system and viewed their file type. *oneforall.jpg* seemed to have some contents so I viewed its exif data.

![](https://cdn.ziomsec.com/ua-high-school/14.webp)

The file had an extension of **jpg** but the file type shown was **png**. So I loaded the file in an online hex editor and viewed the magic headers.

```
exiftool oneforall.jpg
```

![](https://cdn.ziomsec.com/ua-high-school/15.webp)

![](https://cdn.ziomsec.com/ua-high-school/16.webp)

The image had the magic header bytes of **png** type. The typical JPEF header is `FF D8 FF E0 00 10 4A 46 49 46 00 01 00 00 00`. I switched the file headers and downloaded the new image file.

![](https://cdn.ziomsec.com/ua-high-school/17.webp)

Everything seemed fine now.

![](https://cdn.ziomsec.com/ua-high-school/18.webp)

Finally, I tried extracting data from the image. I used the base64 decoded password that I had found in the passphrase.txt file as the password.

![](https://cdn.ziomsec.com/ua-high-school/19.webp)

I had found the credentials of *deku* so I logged in using **ssh**.

```shell
ssh deku@TARGET
```

I captured the user flag from *deku*'s home flag.

![](https://cdn.ziomsec.com/ua-high-school/20.webp)

## Privilege Escalation

I looked at my **sudo** privileges and found I was allowed to execute a bash script. I read the bash script and found it allowed us to execute commands.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/ua-high-school/21.webp)

In Bash, the `eval` command is used to evaluate and execute a string as a shell command.

![](https://cdn.ziomsec.com/ua-high-school/22.webp)

I executed the script and added a new rule in the **sudoers** file allowing my current user to execute all commands as **sudo** without a password.

```shell
deku ALL=NOPASSWD: ALL >> /etc/sudoers
```

![](https://cdn.ziomsec.com/ua-high-school/23.webp)

I verified the changes by viewing my **sudo** privileges.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/ua-high-school/24.webp)

I then executed **bash** with **sudo** and got shell as root. Finally I captured the root flag from */root* directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/ua-high-school/25.webp)

Happy hacking !

---
