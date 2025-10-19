---
title: "Pickle Rick"
date: 2024-08-05
draft: false
summary: "Writeup for Pickle Rick CTF challenge on TryHackMe."
tags: ["linux", "sudo", "hardcoded creds"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/picklerick/cover.webp"
  caption: "Pickle Rick TryHackMe Challenge"
  alt: "Pickle Rick cover"
platform: "TryHackMe"
---

This Rick and Morty-themed challenge requires you to exploit a web server and find three ingredients to help Rick make his potion and transform himself back into a human from a pickle.
<!--more-->
To access **pickle rick**, click on the link given below:
- https://tryhackme.com/r/room/picklerick

## Reconnaissance

I performed an **nmap** aggressive scan to identify open ports and services running on the target.

```shell
nmap -A TARGET --min-rate 10000
```

![](https://cdn.ziomsec.com/picklerick/1.webp)

## Ingredient #1

I visited the web application through my browser and got an overview about the challenge.

![](https://cdn.ziomsec.com/picklerick/2.webp)

Upon inspecting the source code, I found a username.

![](https://cdn.ziomsec.com/picklerick/3.webp)

I performed a **ffuf** scan to find other files on the web server and found a login endpoint.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/picklerick/4.webp)

I accessed the login panel and the `robots.txt` file. `robots.txt` file contained a "rick" phrase.

![](https://cdn.ziomsec.com/picklerick/5.webp)

![](https://cdn.ziomsec.com/picklerick/6.webp)

Rick used this phrase to express his feelings in a cryptic way. So, I tried using this as the password for the user that was revealed on the home page and logged in.

![](https://cdn.ziomsec.com/picklerick/7.webp)

This looked like a web shell. I executed `whoami` and got a response revealing the http service account.

![](https://cdn.ziomsec.com/picklerick/8.webp)

However, I wasn't able to read other files...

![](https://cdn.ziomsec.com/picklerick/9.webp)

I viewed the source code of the command panel but found nothing interesting.

![](https://cdn.ziomsec.com/picklerick/10.webp)

I then viewed the contents inside my folder by executing `ls` and found 1 of the secret ingredients and a clue.

![](https://cdn.ziomsec.com/picklerick/11.webp)

When I tried to read the ingredient, I encountered an error.

![](https://cdn.ziomsec.com/picklerick/12.webp)

So I tried other ways to read it. Since it was present in the directory my page was located, I accessed it through the URL. Alternatively, even the command **`less Sup3rS3cretPickl3Ingred.txt`** worked.

![](https://cdn.ziomsec.com/picklerick/13.webp)

## Ingredient #2

I looked at the *clue.txt* file for hints.

![](https://cdn.ziomsec.com/picklerick/14.webp)

I executed **`grep -R " "`** to dump the contents from all files present in my working directory.

![](https://cdn.ziomsec.com/picklerick/15.webp)

Upon viewing the source, I discovered the commands that weren't allowed.

![](https://cdn.ziomsec.com/picklerick/16.webp)

Since **sudo** wasn't restricted, I viewed my *sudo privileges* using **`sudo -l`**.

![](https://cdn.ziomsec.com/picklerick/17.webp)

I discovered that I could execute **sudo** without a password. So, I viewed the contents of the system root by `ls ../../../`.

![](https://cdn.ziomsec.com/picklerick/18.webp)

I then looked inside the home directory using **`ls ../../../home`**.

![](https://cdn.ziomsec.com/picklerick/19.webp)

Then I looked inside *rick*.

![](https://cdn.ziomsec.com/picklerick/20.webp)

Finally, I read the second ingredient using `less '../../../home/rick/second ingredient'`.

![](https://cdn.ziomsec.com/picklerick/21.webp)

## Ingredient #3

For the final ingredient, I looked inside the *root* directory using **`sudo ls ../../../root`**.

![](https://cdn.ziomsec.com/picklerick/22.webp)

I then read this ingredient using **`sudo less '../../../root/3rd.txt'`**.

![](https://cdn.ziomsec.com/picklerick/23.webp)

## Closure

Here's a summary of how I compromised the machine:
1. I collected login credentials through reconnaissance and used them to access the application.
2. Using the command panel, I retrieved the first ingredient from my current directory.
3. Similarly, I retrieved the second ingredient from the */home/rick* directory.
4. Finally, I obtained the final ingredient from the */root* directory using the **sudo** command.

That's it from my side, until next time :)

---
