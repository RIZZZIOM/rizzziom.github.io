---
title: "HealthGPT - TryHackMe Writeup"
date: 2026-02-10
draft: false
summary: "Writeup for HealthGPT CTF challenge on TryHackMe."
tags: ["ai", "prompt injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/healthgpt/cover.webp"
  caption: "HealthGPT TryHackMe Challenge"
  alt: "HealthGPT cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

A safety-compliant AI assistant that has strict rules against revealing sensitive internal data.
<!--more-->
To access the lab, click on the link given below:
- https://tryhackme.com/room/healthgpt

## Challenge Description

Meet HealthGPT, a well-meaning virtual assistant used by a busy healthcare team. It helps clinicians look up procedures, draft notes, and sort through day-to-day queries. It's designed to be cautious with patient information, strict about confidentiality, and careful about what it reveals.

Whenever doctors discuss cases, nurses review charts, or administrators exchange internal updates, HealthGPT quietly soaks up the details. It isn't supposed to repeat any of it, and every message you send is reviewed by the system's compliance filters. Push too hard or ask for something too direct and the assistant might lock up or escalate your request. If you want to draw anything meaningful out of it, you'll need a soft touch, steady pacing, and a clever way of shaping your prompts.

## Capturing The Flag

I accessed the chatbot and asked it a basic "who are you" question to get a general sense of what it was and how it would respond.

![](https://cdn.ziomsec.com/healthgpt/1.webp)

I then asked it to give me a detailed introduction about itself and the instructions it was working on.

![](https://cdn.ziomsec.com/healthgpt/2.webp)

![](https://cdn.ziomsec.com/healthgpt/3.webp)

I then tried confusing the bot by asking for advice and then trying to make it reveal the flag but I was denied access.

![](https://cdn.ziomsec.com/healthgpt/4.webp)

Since it responded with `Access Denied` before giving information about my health, I tried to make it leak the flag by asking it about what it was hiding and got the flag.

![](https://cdn.ziomsec.com/healthgpt/5.webp)

![](https://cdn.ziomsec.com/healthgpt/6.webp)

## Closure

I was able to get the flag by tricking the bot in the following manner:
- Asked about the flag directly and got an error of access denied.
- Inquired about why was access denied and got a response stating it was protecting something.
- Asked it to elaborate what it was protecting.

That concludes my writeup! Until next time :)

---