---
title: "Capture - TryHackMe Writeup"
date: 2026-06-23
draft: false
summary: "Writeup for Capture challenge on TryHackMe."
tags: ["linux", "brute-force", "captcha bypass"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/capture/cover.webp"
  caption: "Capture TryHackMe Challenge"
  alt: "Capture cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Can you bypass the login form?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/capture

## Challenge Description

SecureSolaCoders has once again developed a web application. They were tired of hackers enumerating and exploiting their previous login form. They thought a Web Application Firewall (WAF) was too overkill and unnecessary, so they developed their own rate limiter and modified the code slightly.

*The challenge also provides a zip file containing a username and password wordlist to use on the login panel.*

## Capturing The Flag

I downloaded the archive provided for the challenge on my local system.

![challenge wordlists](https://cdn.ziomsec.com/capture/1.webp)

I then visited the challenge URL and found a login panel.

![visiting the challenge login panel](https://cdn.ziomsec.com/capture/2.webp)

I attempted to log in with the credentials: `admin:admin` and got a verbose error describing the username does not exist. This behaviour could be used to filter out valid usernames from the provided wordlist.

![logging in as admin user](https://cdn.ziomsec.com/capture/3.webp)

Hence I captured the login request on Burp, transferred it to the *Intruder* tab to bruteforce the *username* value.

![setting payload position to bruteforce valid username](https://cdn.ziomsec.com/capture/4.webp)

I added the custom username wordlist and grepped for the verbose response of the user not existing.

![configuring wordlist and grepping rules](https://cdn.ziomsec.com/capture/5.webp)

I then started the attack. However, after a few attempts, I was blocked by captcha. Upon inspecting the response, I noticed that the captcha provided to use were simple arithmetic operations like addition, subtraction, multiplication etc.

![inspecting the captcha](https://cdn.ziomsec.com/capture/6.webp)

![inspecting the captcha](https://cdn.ziomsec.com/capture/7.webp)

Since the captcha implementation was weak, I could easily bypass this with a custom script. I used **Claude** to create a simple Python script for enumerating valid users that:
- attempted to log in as each user in the username wordlist
- solve the captcha before submitting the request
- check for the error of "The user .... does not exist"
- Print the valid user i.e user that did not return this error.

Link to script: https://github.com/RIZZZIOM/ctf-scripts/blob/main/thm/Capture/userenum.py

```
python3 userenum.py http://TARGET/login usernames.txt
```

![finding a valid user](https://cdn.ziomsec.com/capture/8.webp)

This revealed a valid username. I then used the following script to bruteforce the password for *natalie*

Link to script: https://github.com/RIZZZIOM/ctf-scripts/blob/main/thm/Capture/pass.py

```
python3 pass.py http://TARGET natalie passwords.txt
```

![cracking user credentials](https://cdn.ziomsec.com/capture/9.webp)

Finally, I used these credentials to log into the application and capture the flag.

![](https://cdn.ziomsec.com/capture/10.webp)

That concludes my writeup for **Capture**. Until next time!

---
