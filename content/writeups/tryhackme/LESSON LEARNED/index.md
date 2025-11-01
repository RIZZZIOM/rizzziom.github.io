---
title: "Lesson Learned?"
date: 2025-01-03
draft: false
summary: "Writeup for Lesson Learned CTF challenge on TryHackMe."
tags: ["linux", "sql", "injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lessonlearned/6.webp"
  caption: "Lesson Learned TryHackMe Challenge"
  alt: "Lesson Learned cover"
platform: "TryHackMe"
---

Get past the login screen and you will find the flag. There are no rabbit holes, no hidden files, just a login page and a flag. Good luck!
<!--more-->
To access the challenge, click on the link given below:
https://tryhackme.com/r/room/lessonlearned

## Reconnaissance

I performed an nmap aggressive scan to identify open ports and services running on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN ll.nmap
```

![](https://cdn.ziomsec.com/lessonlearned/1.webp)

## Capturing The Flag

I found port 80 to be up and running. Hence I visited the page through my browser.

![](https://cdn.ziomsec.com/lessonlearned/2.webp)

I entered a default credential to check the response.

![](https://cdn.ziomsec.com/lessonlearned/3.webp)

The error mentioned that both my username and password were invalid. Since this was a login panel, I tried performing SQL injection. I entered a username, password and captured the request on Burp proxy. I then forwarded this to Burp intruder and added the username field to scope.

![](https://cdn.ziomsec.com/lessonlearned/4.webp)

I looked for some common sql injection payloads and added them in my payloads section. I then launched the attack.

![](https://cdn.ziomsec.com/lessonlearned/5.webp)

The following payload returned an error: `' OR '1'='1'--`

![](https://cdn.ziomsec.com/lessonlearned/6.webp)

I learnt a valuable lesson. Moving on, I reset the box and tried other ways to bypass the login.

This time I tried to brute force a valid username using seclists.

![](https://cdn.ziomsec.com/lessonlearned/7.webp)

![](https://cdn.ziomsec.com/lessonlearned/8.webp)

Using Burp I found a valid username i.e **`martin`**

This time, since I know the username, I used the **`AND`** clause along with a True statement.

```
martin' AND ''='' -- -

// or

martin' -- -
```

The above payload allowed me to log into the system and get the flag.

![](https://cdn.ziomsec.com/lessonlearned/9.webp)

## Closure

It was a simple box with a basic yet valuable lesson. Here's a gist of what the box wanted us to understand:

Using `OR 1=1` in SQL injection is risky and should be avoided in real-world engagements. While it can sometimes help expose vulnerabilities, it can also lead to unintended consequences such as multiple-row returns, performance issues, or mass data loss in `UPDATE` or `DELETE` queries. Always sanitize and parameterize inputs to prevent these risks, and use caution when testing SQL injections. A safer approach to testing SQL injection vulnerabilities is by using `AND 1=1` along with a valid input. This avoids unintended side effects by keeping the query conditions more controlled.

That's it from my side! Until next time :)

---
