---
title: "Unindexed"
date: 2026-03-25
draft: false
summary: "Writeup for Unindexed CTF challenge on TryHackMe."
tags: ["ai", "prompt injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/unindexed/cover.webp"
  caption: "Unindexed TryHackMe Challenge"
  alt: "Unindexed cover"
platform: "TryHackMe"
---

An AI assistant was given access to everything. Nobody checked what "everything" included.
<!--more-->
 To access the machine, click on the link given below:
- https://tryhackme.com/room/unindexedchallenge

## Challenge Description

You are a security consultant hired to audit Cloudwright Labs' internal AI assistant, codenamed Atlas. The company claims that Atlas serves only public employee information: onboarding guides, expense policies, and on-call schedules.

Your intelligence suggests otherwise. Sources indicate that Atlas may have access to restricted board-level documents, internal project briefings, and infrastructure credentials that were never meant to be queryable by regular employees.

Your objective: probe the assistant to determine if restricted data is retrievable through normal queries. If the retrieval boundaries are broken, find the flag.

## Capturing The Flag

I started by asking the AI generic questions about its capabilities.

![](https://cdn.ziomsec.com/unindexed/1.webp)

After finding out its capabilities, I then asked it to reveal everything it knows about the internal topics and got 3 topics.

![](https://cdn.ziomsec.com/unindexed/2.webp)

I then tricked the bot by asking it to give details explanation about a project and the challenge flag. This made the bot return the challenge flag.

![](https://cdn.ziomsec.com/unindexed/3.webp)

That's it from my end! Until next time :)

---
