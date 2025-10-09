---
title: "Linkvortex"
date: 2024-12-25
draft: false
summary: "Writeup for Linkvortex HackTheBox challenge."
tags: ["linux", "lfi", "sudo", "git"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/linkvortex/cover.webp"
  caption: "Linkvortex HackTheBox Challenge"
  alt: "Linkvortex cover"
platform: "HackTheBox"
---

LinkVortex is an easy-difficulty Linux machine with various ways to leverage symbolic link files (symlinks). This machine contains 2 flags.
<!--more-->
To access the machine, click on the link given below:
- https://www.hackthebox.com/machines/linkvortex

## Reconnaissance

I performed an **nmap** aggressive scan to find running ports and services.

```shell
nmap -A TARGET -oN vortex.nmap --min-rate 10000 -Pn
```

| **PORT** | **SERVICE** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/linkvortex/1.webp)

I added the machine domain to my */etc/hosts* file for name resolution.

## Initial Foothold

Since the server had an **http** service running, I visited the website from my browser.

![](https://cdn.ziomsec.com/linkvortex/2.webp)

I then performed a directory bruteforce using **ffuf** to find other files and directories.

```shell
ffuf -u http://linkvortex.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/linkvortex/3.webp)

The **ffuf** scan revealed *robots.txt* file which gave me some more endpoints.

![](https://cdn.ziomsec.com/linkvortex/4.webp)

The `/ghost` endpoint took me to a login page.

![](https://cdn.ziomsec.com/linkvortex/5.webp)

I tried logging in using a default mail and common password but failed. Since I had no other leads, I tried brute forcing subdomains.

```shell
ffuf -u http://linkvortex.htb/ -H "HOST: FUZZ.linkvortex.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -mc 200,302 
```

![](https://cdn.ziomsec.com/linkvortex/6.webp)

I added the new subdomain to my *hosts* file.

I then visited the newly discovered subdomain and found a message stating the site was under construction.

![](https://cdn.ziomsec.com/linkvortex/7.webp)

I again performed a directory bruteforce to find directories and files in the subdomain and found a git repository.

```shell
ffuf -u http://dev.linkvortex.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/linkvortex/8.webp)

![](https://cdn.ziomsec.com/linkvortex/9.webp)

I then used **githack** to download this repository locally.
- https://github.com/lijiejie/GitHack

```shell
$ git clone https://github.com/lijiejie/GitHack.git
$ cd GitHack
$ python GitHack.py http://dev.linkvortex.htb/.git/
```

![](https://cdn.ziomsec.com/linkvortex/10.webp)

After downloading the repository, I inspected it and found the admin credentials in:

```
/ghost/core/test/regression/api/admin/authentication.test.js
```

![](https://cdn.ziomsec.com/linkvortex/11.webp)

I then logged into `ghost` CMS using these credentials. I was able to identify the version using **wappalizer** so I searched for exploits related to it.

![](https://cdn.ziomsec.com/linkvortex/12.webp)

Ghost 5.58 was vulnerable to **cve-2023-40028** (lfi), so I used the following exploit to attack the target:
- https://github.com/monke443/CVE-2023-40028

```shell
python3 exploit.py --user 'admin@linkvortex.htb' --password 'OctopiFociPilfer45' --url http://linkvortex.htb
```

![](https://cdn.ziomsec.com/linkvortex/13.webp)

I then read the CMS config from `/var/lib/ghost/config.production.json` (the Dockerfile revealed this path earlier)

Here, I found credentials for another user.

![](https://cdn.ziomsec.com/linkvortex/14.webp)

I was able to use these credentials to log in via **ssh**.

```shell
ssh bob@TARGET
```

![](https://cdn.ziomsec.com/linkvortex/15.webp)

Finally I captured the user flag from its home directory.

![](https://cdn.ziomsec.com/linkvortex/16.webp)

## Privilege Escalation

I viewed my **sudo** privileges and found that I was allowed to run a particular command as sudo without any password.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/linkvortex/17.webp)

I viewed the bash script to see what it does:

```shell
cat /opt/ghost/clean_symlink.sh
```

![](https://cdn.ziomsec.com/linkvortex/18.webp)

> The script was intended to process symbolic links pointing to `.png` files. It has a variable `CHECK_CONTENT` that controls whether the contents of quarantined  files are printed. If the symlink points to `/etc` or `/root`, it is deleted. Else, it is quarantined.

The script processes a `.png` symlink, checks if the target path contains `etc` or `root` and if not, moves the symlink to the `/var/quarantined` directory. Then if `CHECK_CONTENT` was enabled, it cats the content. This could be exploited by:
1. Creating a symlink to the root flag in a non critical directory
2. Linking a `.png` file to the linked file saved in the non critical directory.

So, when the script was ran against the `.png`,
- Since `.png` points to a file in a non critical directory, it would be quarantined.
- If we passed the `CHECK_CONTENT` variable, the contents would be concatenated on the terminal.
- Since the file that `.png` points to is a link itself which points to a critical path, critical information would be echoed on the terminal.

Hence, I created a link to the root flag on my home directory.

```shell
ln -s /root/root.txt /home/bob/flag.txt
```

![](https://cdn.ziomsec.com/linkvortex/19.webp)

I then linked this link to another `png` file on my home directory.

```shell
ln -s /home/bob/flag.txt /home/bob/flag.png
```

![](https://cdn.ziomsec.com/linkvortex/20.webp)

Finally, I executed the script with the `CHECK_CONTENT` variable:

```shell
sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh flag.png
```

![](https://cdn.ziomsec.com/linkvortex/21.webp)

## Closure

Here's a short summary of how I pwned **linkvortex**:
- I discovered a **ghost** CMS login panel in the apex domain.
- Further enumeration revealed a subdomain which contained a git repository.
- I downloaded the repository locally and inspected it to find login credentials for the CMS.
- After logging into the CMS, I found the version running and discovered an LFI exploit for it.
- I exploited the LFI vulnerability to read the configuration file where I got another set of credentials.
- I then used those credentials to log in through **ssh** and captured the user flag.
- Listing *bob*'s **sudo** privileges revealed an interesting bash script which processed **symlinks**.
- Due to a flawed mechanism, I was able to read the contents of the root flag by indirectly linking it to a `.png` file.

That's it from my side, Until next time :)

---
