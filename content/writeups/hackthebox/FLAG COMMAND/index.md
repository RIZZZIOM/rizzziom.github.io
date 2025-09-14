---
title: "Flag Command"
date: 2025-03-05
draft: false
summary: "Writeup for the Flag Command HackTheBox challenge."
tags: ["web", "api"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/flag_command/cover.png"
  caption: "Flag Command HackTheBox Challenge"
  alt: "Flag Command cover"
platform: "HackTheBox"
---

Flag Command is a web based challenge that requires us to play a game of escaping from an alien forest.
<!--more-->
To access the challenge, click on the link given below:
- https://app.hackthebox.com/challenges/Flag%2520Command

I accessed the challenge website and was given a scenario.

![](https://cdn.ziomsec.com/flag_command/1.png)

I started the game and tried selecting few options to understand the working of the application.

```
start
HEAD NORTH
```

![](https://cdn.ziomsec.com/flag_command/2.png)

```
FOLLOW A MYSTERIOUS PATH
SET UP CAMP
```

![](https://cdn.ziomsec.com/flag_command/3.png)

After few choices, my player died and I had to restart the game. The game took fixed prompts as inputs and threw error when something unexpected was entered.

![](https://cdn.ziomsec.com/flag_command/4.png)

So I reloaded the page and viewed my **Network** tab to see the requests made by the application.

Here, I found the `/api/options` endpoint that fetched the available options.

![](https://cdn.ziomsec.com/flag_command/5.png)

The response for the request returned all available options in the game where I found a secret command.

![](https://cdn.ziomsec.com/flag_command/6.png)

I entered the secret command and captured the flag.

![](https://cdn.ziomsec.com/flag_command/7.png)

That's it from my side! Until next time :)

---
