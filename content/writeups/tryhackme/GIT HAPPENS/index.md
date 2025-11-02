---
title: "Git Happens"
date: 2024-08-29
draft: false
summary: "Writeup for Git Happens CTF challenge on TryHackMe."
tags: ["git"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/githappens/cover.webp"
  caption: "Git Happens TryHackMe Challenge"
  alt: "Git Happens cover"
platform: "TryHackMe"
---

Boss wanted me to create a prototype, so here it is! We even used something called "version control" that made deploying this really easy!
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/r/room/githappens

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and services running on the machine.

```shell
nmap -A -p- TARGET -oN git.nmap --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |

![](https://cdn.ziomsec.com/githappens/1.webp)

## Capturing The Flag

The scan revealed port 80 to be up and running. It also found a directory `/.git` on the website. I visited the website from my browser.

![](https://cdn.ziomsec.com/githappens/2.webp)

To find more directories, I performed web fuzzing using **ffuf**

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
```

![](https://cdn.ziomsec.com/githappens/3.webp)

The fuzz scan revealed another page `dashboard.html`. I then navigated to `/.git` and found contents of a git repository. 

![](https://cdn.ziomsec.com/githappens/4.webp)

In order to make working with it more convinient, I used **[gittools](https://github.com/internetwache/GitTools)**.
It contains a script called **gitdumper**. This tool can be used to download as much as possible from the found `.git` repository from webservers which do not have directory listing enabled. I used it to copy the contents from the site to my local machine.

```shell
./gitdumper.sh http://TARGET/.git/ /PATH/TO/SAVE
```

![](https://cdn.ziomsec.com/githappens/5.webp)

![](https://cdn.ziomsec.com/githappens/6.webp)

I viewed the status of the repo using **`git status`** and found a lot of files had been deleted.

```shell
git status
```

![](https://cdn.ziomsec.com/githappens/7.webp)

I then viewed logs to find information about the commits that were made.

```shell
git log
```

![](https://cdn.ziomsec.com/githappens/8.webp)

![](https://cdn.ziomsec.com/githappens/9.webp)

The commit with **`Made the login page, boss!`** comment looked interesting. So I viewed the information that was sent in that commit.

```shell
git show HASH
```

![](https://cdn.ziomsec.com/githappens/10.webp)

Here I found the hardcoded credentials for the **login** page.

![](https://cdn.ziomsec.com/githappens/11.webp)

I entered these credentials and tried logging in but I didn't get any response from the website. So I revisited the commit to look for more information.

![](https://cdn.ziomsec.com/githappens/12.webp)

![](https://cdn.ziomsec.com/githappens/13.webp)

I found another message that I had skipped previously. The flag was the password used to log in. So I submitted the password on **tryhackme** and solved the challenge.

Thats it from side :)
Until next time

---
