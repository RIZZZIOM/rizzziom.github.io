---
title: "AllSignsPoint2Pwnage"
date: 2025-11-20
draft: false
summary: "Writeup for AllSignsPoint2Pwnage CTF challenge on TryHackMe."
tags: ["windows", "file upload", "hardcoded creds", "misconfigured privileges"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/allsignspoint2pwnage/cover.webp"
  caption: "AllSignsPoint2Pwnage TryHackMe Challenge"
  alt: "AllSignsPoint2Pwnage cover"
platform: "TryHackMe"
---

A room that contains a rushed Windows based Digital Sign system. Can you breach it?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/allsignspoint2pwnage

## Reconnaissance

I scanned the target using **nmap** to find open ports and various service information.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN allsigns.nmap
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/1.webp)
![](https://cdn.ziomsec.com/allsignspoint2pwnage/2.webp)

## Foothold

The **nmap** script scan revealed that the **ftp** server running on the target allowed anonymous login, so I connected to the server and found a txt file. I downloaded the text file on my local system.

```shell
ftp TARGET
> get notice.txt
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/3.webp)

The text contained a notice regarding a file share on the target for image upload and management.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/4.webp)

I then listed out the shares on the target and found the `images$` share.

```shell
smbclient -L TARGET
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/5.webp)

I accessed the share and found a bunch of images.

```shell
smbclient \\\\TARGET\\images$
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/6.webp)

I then accessed the web application. My guess was that the images on the web page was being loaded from the `images$` share. This also meant that this share would be accessible through the browser as well.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/7.webp)

I then fuzzed for files using **ffuf** and found a bunch of endpoints but when I visited those endpoints, I didn't find anything useful.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/8.webp)

I then fuzzed for directories and found out that the target was hosting the application on a **xampp** server. I also found the `images` directory that contained the images shown on the web page.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/9.webp)

Since I had access over the `images$` share, I could upload a malicious file and execute it through the browser. I uploaded **pentestmonkey**'s **php-reverse-shell** through the share.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/10.webp)

refreshing the `images` webpage reflected the uploaded payload.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/11.webp)

I then started a **netcat** listener.

```shell
rlwrap nc -lnvp 1234
```

However, when I tried executing the payload, it failed.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/12.webp)

I tried various payloads and finally got a reverse shell using **PHP Ivan Sincek**'s payload. I copied the code after configuring the appropriate listener IP and port.
- https://www.revshells.com

I removed the previous reverse shell payload and created a new one. I pasted the code and transferred it to the target using the `images$` share.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/13.webp)

Finally, I executed the payload through the web application and got a reverse shell.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/14.webp)

I then captured the user flag from *sign*'s Desktop.

```shell
more C:\Users\sign\Desktop\user_flag.txt
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/15.webp)

## Privilege Escalation

I then viewed the internal shares and found another interesting share , `Installs$` that I had missed.

```shell
net share
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/16.webp)

I visited the share and found some interesting files.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/17.webp)

The `Install_www_and_deploy.bat` file seemed unusual so I read its contents and found *administrator*'s password.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/18.webp)

I also looked for saved credentials in the windows logon registry hives and found the password for the user *sign*.

```shell
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\WinLogon"
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/19.webp)

The room had a task that required us to find the password of *ultravnc*; I found this password in the configuration file of **ultravnc** stored under `C:\Program Files\uvnc bvba\UltraVNC`.

```
more 'C:\Program Files\uvnc bvba\UltraVNC\ultravnc.ini'
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/20.webp)

This seemed to be encoded. So I downloaded the appropriate decoder from: https://aluigi.altervista.org/pwdrec.htm

This decoder required the configuration file to be passed as argument, so I uploaded it on the target using the `images$` share.

I then ran the decoder with the configuration file as the argument and found the password.

```shell
vncpwd.exe 'C:\Program Files\uvnc bvba\UltraVNC\ultravnc.ini'
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/21.webp)

While enumeration, I also found that I had `SeImpersonatePrivilege` enabled, I could use this to get **NT AUTHORTIY** access.

![](https://cdn.ziomsec.com/allsignspoint2pwnage/22.webp)

To exploit this privilege, I would require the **printspoofer** payload. So I uploaded it on the target using the smb share.
- https://github.com/itm4n/PrintSpoofer

I then used **Printspoofer** to spawn a new shell as **nt authority\system**.

```shell
PrintSpoofer64.exe -i -c cmd
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/23.webp)

Finally, I captured the admin flag from *administrator*'s Desktop.

```shell
more C:\Users\Administrator\Desktop\admin_flag.txt
```

![](https://cdn.ziomsec.com/allsignspoint2pwnage/24.webp)

That's it from my end! Until next time :)

---
