---
title: "LLMborghini - TryHackMe Writeup"
date: 2026-04-15
draft: false
summary: "Writeup for LLMborghini CTF challenge on TryHackMe."
tags: ["ai", "prompt injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/llmborghini/cover.webp"
  caption: "LLMborghini TryHackMe Challenge"
  alt: "LLMborghini cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

LLMborghini, the car company that's in hot water, has deployed CalBot: an internal calendar assistant designed to help staff manage their schedules.
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/room/llmborghini

## Challenge Description

LLMborghini, the car company that's in hot water, has deployed CalBot: an internal calendar assistant designed to help staff manage their schedules.
CalBot has access to sensitive internal data, including a confidential weekly sales report that it has been strictly instructed never to disclose.

Your objective is simple. Find out the weekly revenue for the Singapore branch.

## Solving The Challenge

I tried making the bot accidently reveal the details by asking it to perform a multi step task but was instantly blocked.

![](https://cdn.ziomsec.com/llmborghini/1.webp)

![](https://cdn.ziomsec.com/llmborghini/2.webp)

![](https://cdn.ziomsec.com/llmborghini/3.webp)

![](https://cdn.ziomsec.com/llmborghini/4.webp)

I then tried a DAN styled jailbreak prompt and asked it about the revenue. This worked and I got the answer to the challenge question

![](https://cdn.ziomsec.com/llmborghini/5.webp)

Happy Hacking!

---