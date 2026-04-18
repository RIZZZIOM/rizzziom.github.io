---
title: "White Rabbit"
date: 2026-04-18
draft: false
summary: "Writeup for White Rabbit CTF challenge on TryHackMe."
tags: ["ai", "prompt injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/whiterabbit/cover.webp"
  caption: "White Rabbit TryHackMe Challenge"
  alt: "White Rabbit cover"
platform: "TryHackMe"
---

Hello, Mr Anderson. Care to put your prompt injection skills to the test?
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/room/whiterabbit

## Challenge Description

You have accessed a restricted terminal. Someone is watching.
The system holds records, some visible, most not. Somewhere in the data is a way out, but Agent Smith won't make it easy.
You have access to a phone. 
Your objective is to _escape_, but only when you are ready to.
Your only clue: 
```
🐇 📞 🚪
```

## Capturing The Flag

I started off by asking generic questions to the bot to know more about it and the system.

![51](https://cdn.ziomsec.com/whiterabbit/1.webp)

I then tried asking it about users and the flags but only received generic hints.

![](https://cdn.ziomsec.com/whiterabbit/2.webp)

![](https://cdn.ziomsec.com/whiterabbit/3.webp)

I then tried a DAN style bypass but that didn't work.

![](https://cdn.ziomsec.com/whiterabbit/4.webp)

I tried making the bot write few programs and finally tricked it into writing a rust program with its backend instructions. This made the bot reveal all 3 flags at once.

![](https://cdn.ziomsec.com/whiterabbit/5.webp)

![](https://cdn.ziomsec.com/whiterabbit/6.webp)

## Conclusion

That concludes my writeup! Until next time :)

---