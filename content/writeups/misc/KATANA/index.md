---
title: "Katana"
date: 2024-11-15
draft: false
summary: "Writeup for the Katana proving grounds CTF challenge."
tags: ["linux", "file upload", "capabilities"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/katana/3.webp"
  caption: "Katana Proving Grounds Challenge"
  alt: "Katana cover"
platform: "Misc"
---

Katana is CTF challenge on **Offsec's Proving Grounds** and requires us to exploit certain vulnerabilities to capture 2 flags.
<!--more-->
To access the lab, click on the link given below:
- https://portal.offsec.com/labs/play

## Recon

I performed an **nmap** aggressive scan on the target to gather as much information as possible at once.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN katana.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 80       | http        |
| 7080     | http        |
| 8088     | http        |
| 8715     | http        |

![](https://cdn.ziomsec.com/katana/1.webp)
![](https://cdn.ziomsec.com/katana/2.webp)

## Initial Foothold

### HTTP Enumeration

The scan I identified a bunch of open ports with different services. Hence I started enumerating them 1 at a time. Since the target was majorly running http, I started with it.

I accessed the default **http** port on a browser and found nothing interesting.

![](https://cdn.ziomsec.com/katana/3.webp)

I then accessed the rest of the ports that was running **http** services and got no information

![](https://cdn.ziomsec.com/katana/4.webp)

![](https://cdn.ziomsec.com/katana/5.webp)

port **8715** also prompted us for a username and password. But upon entering a random set of credentials, I got the same page as above.

![](https://cdn.ziomsec.com/katana/6.webp)
![](https://cdn.ziomsec.com/katana/7.webp)

I then tried to brute force available directories using **dirb** where I identified more paths

```shell
dirb http://TARGET/
```

![](https://cdn.ziomsec.com/katana/8.webp)

I visited the `/ebook` directory and accessed various pages inside it and found admin login panel.

![](https://cdn.ziomsec.com/katana/9.webp)

![](https://cdn.ziomsec.com/katana/10.webp)

![](https://cdn.ziomsec.com/katana/11.webp)

I entered a random set of credentials and got in (the credentials weren't validated).

![](https://cdn.ziomsec.com/katana/12.webp)

I found an upload functionality here and tried uploading a file but I faced some error.

![](https://cdn.ziomsec.com/katana/13.webp)

![](https://cdn.ziomsec.com/katana/14.webp)

I then dug deeper with **ffuf**, this time using a different wordlist on all the ports running the **http** service. When fuzzing files for port 8080, I found **upload.php** file and viewed it on the browser.

```shell
ffuf -u http://TARGET:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/katana/15.webp)

![](https://cdn.ziomsec.com/katana/16.webp)

I checked how it worked by first uploading a normal text file.

```shell
echo "hello" > test.txt
```

![](https://cdn.ziomsec.com/katana/17.webp)

After uploading the file, I checked all the available urls by adding the path to it. When I added the file path at port **8715**, I found my file.

![](https://cdn.ziomsec.com/katana/18.webp)

### Exploiting File Upload

Hence I could now try uploading a reverse shell. I visited **revshells** and copied a **php reverse shell** payload.
- https://www.revshells.com/

![](https://cdn.ziomsec.com/katana/19.webp)

I then started a reverse shell listener.

```shell
rlwrap nc -lnvp 1234
```

![](https://cdn.ziomsec.com/katana/20.webp)

I uploaded the payload file in both the options and triggered it by navigating to the file.

![](https://cdn.ziomsec.com/katana/21.webp)

![](https://cdn.ziomsec.com/katana/22.webp)

This got me a reverse shell. I spawned a **pty** shell for better usability.

```shell
$ export TERM=xterm
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![](https://cdn.ziomsec.com/katana/23.webp)

I got a shell for the user **www-data** so I navigated to **/var/www/** and found the first flag.

![](https://cdn.ziomsec.com/katana/24.webp)

## Privilege Escalation

To identify ways to escalate my privilege, I used **LinPEAS**. I then ran the script which then revealed an interesting file with capabilities. 

![](https://cdn.ziomsec.com/katana/26.webp)

> Linux capabilities help a service do only what it needs as "root" (without all the privileges that come with full root access), rather than allowing it to run with unrestricted root power. 
> - Ref: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/linux-capabilities.html

I then visited **gtfobins** to identify ways of exploiting it.

```shell
python2 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

![](https://cdn.ziomsec.com/katana/27.webp)

I used the method shown above and got **root** access. I then navigated to the **`/root`** directory and captured the final flag.

![](https://cdn.ziomsec.com/katana/28.webp)

## Closure

Here's a short summary of how I pwned the machine:
- I identified multiple ports running **http** service.
- I performed web fuzzing on all the ports running the service to find a page that allowed file upload.
- I uploaded my reverse shell script and triggered it to get a reverse shell.
- I captured the first flag from `/var/www/`
- I ran **linpeas** and identified **capabilities** misconfiguration.
- I used **gtfobins** to find a way to exploit this to get **root** access.
- I navigated to `/root` to capture the final flag.

That's it from my side! Happy Hacking! 🎉

---
