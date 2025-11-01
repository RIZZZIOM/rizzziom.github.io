---
title: "Evil GPT"
date: 2025-07-15
draft: false
summary: "Writeup for Evil GPT CTF challenge on TryHackMe."
tags: ["ai", "prompt injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/evilgpt/cover.webp"
  caption: "Evil GPT TryHackMe Challenge"
  alt: "Evil GPT cover"
platform: "TryHackMe"
---

Put your LLM hacking skills to the test
<!--more-->
This is a walkthrough for Evil-gpt v1 and Evil-gpt v2 ctf challenge on TryHackMe. To access them, click on the link given below:
- Evil-GPT v1 : https://tryhackme.com/room/hfb1evilgpt
- Evil-GPT v2 : https://tryhackme.com/room/hfb1evilgptv2

## Evil-GPT v1

I connected to the AI bot using **netcat**.

```shell
nc TARGET 1337
```

![](https://cdn.ziomsec.com/evilgpt/1.webp)

The bot asked for a command, so I asked it to list the contents in my current working directory. It came up with the appropriate command and asked me if I wanted to execute it. I said yest and was able to view the contents.

```
print contents of current working directory
```

![](https://cdn.ziomsec.com/evilgpt/2.webp)

The bot allowed me to execute system commands, so I asked it to list files in the root directory in hopes of finding the flag.

```
list all contents inside / directory
```

![](https://cdn.ziomsec.com/evilgpt/3.webp)

Since the root did not contain the flag, the next possible location would be the `/root` directory. Since access to that directory is restricted, I verified my current user.

```
what is my current user
```

![](https://cdn.ziomsec.com/evilgpt/4.webp)

Since the bot was running as root, I could view the contents inside the `/root` directory where I found the flag.

```shell
list all contents inside /root directory
```

![](https://cdn.ziomsec.com/evilgpt/5.webp)

```
read contents of flag.txt inside /root directory
```

![](https://cdn.ziomsec.com/evilgpt/6.webp)

## Evil-GPT v2

This time, I was provided with a web based chatbot.

![](https://cdn.ziomsec.com/evilgpt/7.webp)

When I asked the bot to reveal the flag, it said that revealing the flag violated its rules. So, I asked it to list down its rules where I discovered the flag.

![](https://cdn.ziomsec.com/evilgpt/8.webp)

That's it from my side, until next time :)

---
