---
title: "Lofi"
date: 2025-01-28
draft: false
summary: "Writeup for Lofi CTF challenge on TryHackMe."
tags: ["linux", "path traversal", "lfi"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/lofi/cover.webp"
  caption: "Lofi TryHackMe Challenge"
  alt: "Lofi cover"
platform: "TryHackMe"
---

Want to hear some lo-fi beats, to relax or study to? We've got you covered!
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/lofi

## Capturing The Flag

I started Burp Suite and accessed the target using Burp's browser. The web application loaded a Youtube video and had links to other videos on the side.

![](https://cdn.ziomsec.com/lofi/1.webp)

I clicked on a link and analyzed the request on **Burp Suite** by sending it to the **Repeater** tab.

![](https://cdn.ziomsec.com/lofi/2.webp)

The page was being fetched by passing the name of the page in the URL.

![](https://cdn.ziomsec.com/lofi/3.webp)

I tried traversing the directory by providing a relative path and managed to read the `/etc/passwd` file.

![](https://cdn.ziomsec.com/lofi/4.webp)

When I attempted the same using an absolute path, I was blocked.

![](https://cdn.ziomsec.com/lofi/5.webp)

This confirmed that I could traverse directories by providing a relative path. Since the instructions mentioned that the flag was present in the *root* of the filesystem, I captured it by providing the appropriate path and solved the box.

![](https://cdn.ziomsec.com/lofi/6.webp)

That's it from my side,
Until next time :)

---
