---
title: "Funbox Easy Enum"
date: 2024-12-1
draft: false
summary: "Writeup for the Funbox Easy Enum proving grounds CTF challenge."
tags: ["linux", "sudo", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/funboxeasyenum/2.webp"
  caption: "Funbox Easy Enum Proving Grounds Challenge"
  alt: "Funbox Easy Enum cover"
platform: "Misc"
---

**Fun Box Easy Enum** is an easy CTF challenge on **Proving Grounds** that require us to capture 2 flags hidden in it.
<!--more-->
To access the lab, click on the link given below:
- https://portal.offsec.com/labs/play

## Recon

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET -oN easyenum.nmap --min-rate 10000 -Pn
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/funboxeasyenum/1.webp)

## Initial Foothold

The scan identified **http** service to be up and running so I accessed it through my browser and found a default **apache** landing page.

![](https://cdn.ziomsec.com/funboxeasyenum/2.webp)

I used **ffuf** to find hidden directories and files.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordists/seclists/Discovery/Web-Content/big.txt
```

![](https://cdn.ziomsec.com/funboxeasyenum/3.webp)

I tried accessing the **robots.txt** file but found nothing interesting.

```shell
curl http://TARGET/robots.txt
```

![](https://cdn.ziomsec.com/funboxeasyenum/4.webp)

Another page identified while fuzzing was **phpmyadmin**, so I tried accessing it and used default credentials for logging in.

![](https://cdn.ziomsec.com/funboxeasyenum/5.webp)

Since the default credentials didn't work, I tried digging deeper by enumerating **file extensions** using **ffuf**. I tried common extensions like **`.js`**, **`.php`**, **`.asp`**, **`.aspx`** and found a file with **`.php`** extension.

```shell
ffuf -u http://TARGET/FUZZ.php -w /usr/share/wordists/seclists/Discovery/Web-Content/big.txt
```

![](https://cdn.ziomsec.com/funboxeasyenum/6.webp)

I accessed it on the browser and found it to be a graphical user interface for the **`/var/www/html`** directory. It allowed various operations on the files present inside like, change permissions, delete, add, rename etc.

![](https://cdn.ziomsec.com/funboxeasyenum/7.webp)

Here I found the first flag and read it using the available functions.

![](https://cdn.ziomsec.com/funboxeasyenum/8.webp)

![](https://cdn.ziomsec.com/funboxeasyenum/9.webp)

Next I downloaded the **php reverse shell** payload from **pentestmonkey** on my local system.
- https://github.com/pentestmonkey/php-reverse-shell

I modified the payload to add my listening address and port and uploaded it on the target.

![](https://cdn.ziomsec.com/funboxeasyenum/10.webp)

I gave it read, write and execute permissions for owner, group and others.

![](https://cdn.ziomsec.com/funboxeasyenum/11.webp)

Finally I triggered the payload by attempting to access it and got a reverse shell.

![](https://cdn.ziomsec.com/funboxeasyenum/12.webp)

![](https://cdn.ziomsec.com/funboxeasyenum/13.webp)

I spawned a **pty** shell using **python** and exported my terminal for better usability.

```shell
$ export TERM=xterm
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

## Privilege Escalation

### Shell As `Oracle`

I transferred the **linux smart enumeration** script from my system to the target to identify misconfigurations that could help me escalate my privilege.
- https://github.com/diego-treitos/linux-smart-enumeration

![](https://cdn.ziomsec.com/funboxeasyenum/14.webp)

I ran the script and found a hash for **oracle** user.

```shell
$ chmod +x lse.sh
$ ./lse.sh
```

![](https://cdn.ziomsec.com/funboxeasyenum/15.webp)

I referred to **haschat** example hashes and found the hash type - **md5**.
- https://hashcat.net/wiki/doku.php?id=example_hashes

![](https://cdn.ziomsec.com/funboxeasyenum/16.webp)

I copied the hash on my local system and cracked it using **hashcat** with **rockyou.txt** wordlist.

```shell
$ echo '<HASH>' > myhash
$ hashcat -m 500 myhash /usr/share/wordlists/rockyou.txt
```

![](https://cdn.ziomsec.com/funboxeasyenum/17.webp)

I then switched to **oracle** using the cracked password.

```shell
su oracle
```

![](https://cdn.ziomsec.com/funboxeasyenum/18.webp)

I tried looking around but found nothing interesting so I switched back to the **www-data** user.

### Shell As `Karla`

I remembered finding a **phpmyadmin** page when I was **fuzzing** the web directories so I look for interesting files in it. The default path for **phpmyadmin** is **`/etc/phpmyadmin`** so I navigated to it Here I searched multiple config files and fond a username and password in one of the files.

```shell
$ cd /etc/phpmyadmin
$ cat config-db.php
```

![](https://cdn.ziomsec.com/funboxeasyenum/19.webp)

I tried a password spray attack on the users that were present in the **`/home`** directory and got access to **karla**. 

### Exploiting Sudo Permissions

Upon logging in, I got a message regarding **sudo**. So I tried looking at the **sudo** privileges **karla** had. 

```shell
sudo -l
```

![](https://cdn.ziomsec.com/funboxeasyenum/20.webp)

**Karla** had the permission to run all commands as **sudo** without password. So I used it to spawn a **bash** shell. Once I became the **root** user, I navigated to the **`/root`** directory and captured the final flag.

```shell
$ sudo /bin/bash
$ cd /root
$ cat proof.txt
```

![](https://cdn.ziomsec.com/funboxeasyenum/21.webp)

## Closure

Here's a summary of how I pwned the machine:
- I performed web fuzzing to find a **php** file that provided a GUI for working with contents inside the **`/var/www/html`** directory.
- I found the first flag in this directory.
- I used this interface to upload my **php reverse shell** script.
- I got a reverse shell by triggering the payload. 
- I did further enumeration and found the hash of one of the user's. I cracked the hash but then found nothing interesting upon switching users.
- I investigated the **phpmyadmin** file for juicy information and found a set of credentials in one of the files inside **`/etc/phpmyadmin`**.
- I tried switching users using this password and got access to **Karla**
- **Karla** was authorized to run **sudo** with all commands so I used this to spawn a **bash** shell as **root**.
- Once I became **root** user, I navigated to the **/root** directory and captured the final flag.

That's it from my side! Happy Hacking ;)

---
