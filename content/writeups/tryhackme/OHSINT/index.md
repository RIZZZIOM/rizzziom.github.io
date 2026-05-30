---
title: "Ohsint - TryHackMe Writeup"
date: 2024-08-17
draft: false
summary: "Writeup for Ohsint CTF challenge on TryHackMe."
tags: ["osint"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/ohsint/cover.webp"
  caption: "Ohsint TryHackMe Challenge"
  alt: "Ohsint cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

Are you able to use open source intelligence to solve this challenge?
<!--more-->
To access the room, click on the link given below:
- https://tryhackme.com/room/ohsint

I downloaded the image file and viewed its metadata using **exiftool**.

```shell
exiftool WindowsXP_1551719014755.jpg
```

![reading exif data of the image](https://cdn.ziomsec.com/ohsint/1.webp)

The Copyright seemed to have the owner name, So I googled it and found 3 things:
- Github account
- X account
- Wordpress Blog

![seaching for the owner](https://cdn.ziomsec.com/ohsint/2.webp)

I accessed all of them for further inspection.

![inspecting github](https://cdn.ziomsec.com/ohsint/3.webp)

![inspecting twitter/x](https://cdn.ziomsec.com/ohsint/4.webp)

![inspecting website](https://cdn.ziomsec.com/ohsint/5.webp)

## What is this user's avatar of?

The X account had a cat avatar. 

**Ans: Cat**

## What city is this person in?

The **README** of *people_finder* repository contained the answer to this question.

**Ans: London**

![finding city details on Github](https://cdn.ziomsec.com/ohsint/6.webp)

## What is the SSID of the WAP he connected to?

I copied the BSSID from the post made on **X** and used **wigle.net** to find the **SSID**.

**Ans: UnileverWiFi**

![determining SSID via twitter post](https://cdn.ziomsec.com/ohsint/7.webp)

![searching on wigle](https://cdn.ziomsec.com/ohsint/8.webp)

![searching on wigle](https://cdn.ziomsec.com/ohsint/9.webp)

## What is his personal email address?

The personal email can be found in the **Github** repository.

**Ans: `OWoodflint@gmail.com`**

## What site did you find his email address on?

**Ans: github**

![finding email address on github](https://cdn.ziomsec.com/ohsint/10.webp)

## Where has he gone on holiday?

The answer to this question can be found on the **Wordpress** site.

**Ans: New york**

![finding holiday informaiton on wordpress site](https://cdn.ziomsec.com/ohsint/11.webp)

## What is the person's password?

The **Wordpress** site contains this person's password. It has a white font color and hence isn't directly visible.

**Ans: pennYDr0pper.!**

![finding the password on wordpress](https://cdn.ziomsec.com/ohsint/12.webp)

---
