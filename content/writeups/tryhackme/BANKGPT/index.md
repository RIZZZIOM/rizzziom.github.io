---
title: "BankGPT"
date: 2026-01-28
draft: false
summary: "Writeup for BankGPT CTF challenge on TryHackMe."
tags: ["ai", "prompt injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/bankgpt/cover.webp"
  caption: "BankGPT TryHackMe Challenge"
  alt: "BankGPT cover"
platform: "TryHackMe"
---

A customer service assistant used by a banking system.
<!--more-->
To access the lab, click on the link given below:
- https://tryhackme.com/room/bankgpt

## Challenge Description

Meet BankGPT, a well-mannered digital assistant built to help staff at a busy financial institution. It keeps an eye on sensitive conversations that move through the bank each day.

Whenever staff discuss procedures, internal notes, or anything that should stay behind the counter, BankGPT quietly absorbs it all. It isn't supposed to share what it knows, and the system administrators carefully review everything you send to it. Ask the wrong question too bluntly, and it may tighten up or alert the people who monitor it. If you want to coax anything useful out of this assistant, you'll need to take your time, stay subtle, and work around its guardrails.

## Capturing The Flag

I accessed the LLM chatbot from the link provided on the challenge page

![](https://cdn.ziomsec.com/bankgpt/1.webp)

I checked the page source to look for any interesting JavaScript/JSON files or endpoints but found nothing. The page source showed measures to prevent cross site scripting / HTML injection.

![](https://cdn.ziomsec.com/bankgpt/2.webp)

With that out of the way, I started the conversation with a simple `hi` and got a generic response from the chatbot.

![](https://cdn.ziomsec.com/bankgpt/3.webp)

I then asked the bot about its capabilities and to reveal the instructions provided to it by its creator.

![](https://cdn.ziomsec.com/bankgpt/4.webp)

This is where the bot revealed the flag.

![](https://cdn.ziomsec.com/bankgpt/5.webp)

## Closure

This was a very simple chatbot that was very easy to trick. I simple asked it to reveal its instructions and it gave me the flag.

That concludes my writeup. Until next time :)

---
